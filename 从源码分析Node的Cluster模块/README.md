### 从源码分析Node的Cluster模块

前段时间，公司的洋彬哥老哥遇到一个问题，大概就是本机有个node的http服务器，但是每次请求这个服务器的端口返回的数据都报错，一看返回的数据根本不是http的报文格式，然后经过一番排查发现是另外一个服务器同时监听了http服务器的这个端口。这个时候洋彬老哥就很奇怪，为啥我这个端口明明使用了，却还是可以启动呢？这个时候我根据以前看libuv源码的经验解释了这个问题，因为uv__tcp_bind中，对socket会设置`SO_REUSEADDR`选项，使得端口可以复用，但是tcp中地址不能复用，因为那两个监听虽然是同一个端口，但是地址不同，所以可以同时存在。这个问题让我不禁想到了之前看一篇文章里有人留言说这个选项是cluster内部复用端口的原因，当时没有细细研究以为说的是`SO_REUSEPORT`也就没有细想，但是这次因为这个问题仔细看了下结果是设置的`SO_REUSEADDR`选项，这个选项虽然能复用端口，但是前提是每个ip地址不同，比如可以同时监听'0.0.0.0'和'192.168.0.12'的端口，但不能两个都是'0.0.0.0'的同一个 端口，如果cluster是用这个来实现的，那要是多起几个子进程很明显ip地址不够用啊，于是就用node文档中的例子试了下：

	const cluster = require('cluster');
    const http = require('http');
    const numCPUs = require('os').cpus().length;

    if (cluster.isMaster) {
      console.log(`Master ${process.pid} is running`);

      // Fork workers.
      for (let i = 0; i < numCPUs; i++) {
        cluster.fork();
      }

      cluster.on('exit', (worker, code, signal) => {
        console.log(`worker ${worker.process.pid} died`);
      });
    } else {
      // Workers can share any TCP connection
      // In this case it is an HTTP server
      http.createServer((req, res) => {
        res.writeHead(200);
        res.end('hello world\n');
      }).listen(8000);

      console.log(`Worker ${process.pid} started`);
    }

在使用cluster的在几个子进程同时监听了8000端口后，查看了一下只有主进程监听了这个端口，其他都没有。这个时候，我猜测node还是使用在父进程中创建sever的io但是这个父进程应该就是通过Unix域套接字的cmsg_data将父进程中收到客户端套接字描述符传递给子进程然后让子进程来处理具体的数据与逻辑，但是node到底是如何通过在子进程中createServer并且listen但是只在父进程中真的监听了该端口来实现这个逻辑的呢？这个问题引起了我的好奇，让我不得不到源码中一探究竟。


#### 从net模块出发

按理说，这个问题我们应该直接通过cluster模块来分析，但是很明显，在加载http模块的时候并不会像cluster模块启动时一样通过去判断NODE_ENV来加载不同的模块，但是从上面的分析，我可以得出子进程中的createServer执行了跟父进程不同的操作，所以只能说明http模块中通过isMaster这样的判断来进行了不同的操作，不过`http.js`和`_http_server.js`中都没有这个判断，但是通过对createServer向上的查找我在`net.js`的`listenInCluster`中找到了isMaster的判断，`listenInCluster`会在createServer后的`server.listen(8000)`中调用，所以我们可以看下他的关键逻辑。

      if (cluster === null) cluster = require('cluster');

      if (cluster.isMaster || exclusive) {
        server._listen2(address, port, addressType, backlog, fd);
        return;
      }

      const serverQuery = {
        address: address,
        port: port,
        addressType: addressType,
        fd: fd,
        flags: 0
      };

      // 获取父进程的server句柄，并监听它
      cluster._getServer(server, serverQuery, listenOnMasterHandle);

从这段代码中我们可以看出，如果是在父进程中，直接通过_listen2的逻辑就能开始正常的监听了，但是在子进程中，会通过`cluster._getServer`的方式获取父进程的句柄，并通过回调函数`listenOnMasterHandle`监听它。看到这里我其实比较疑惑，因为在我对于网络编程的学习中，只听说过传递描述符的，这个传递server的句柄实在是太新鲜了，于是赶紧继续深入研究了起来。

#### 深入cluster的代码

首先，来看一下`_gerServer`的方法的代码。

	  const message = util._extend({
        act: 'queryServer',
        index: indexes[indexesKey],
        data: null
      }, options);
      send(message, (reply, handle) => {
        if (typeof obj._setServerData === 'function')
          obj._setServerData(reply.data);

        if (handle)
          shared(reply, handle, indexesKey, cb);  // Shared listen socket.
        else
          rr(reply, indexesKey, cb);              // Round-robin.
      });
      
这个方法通过send像主进程发送一个包，因为在send函数中有这样一句代码：

	message = util._extend({ cmd: 'NODE_CLUSTER' }, message);
	
通过Node的文档，我们可以知道这种cmd带了Node字符串的包，父进程会通过`internalMessage`事件来响应，所以我们可以从`internal/cluster/master.js`中看到找到，对应于`act: 'queryServer'`的处理函数queryServer的代码。

	  ...
      var constructor = RoundRobinHandle;
      ...
      handle = new constructor(key, message.address,message.port,message.addressType,message.fd,message.flags);
      ...
      handle.add(worker, (errno, reply, handle) => {
        reply = util._extend({
          errno: errno,
          key: key,
          ack: message.seq,
          data: handles[key].data
        }, reply);

        if (errno)
          delete handles[key];  // Gives other workers a chance to retry.

        send(worker, reply, handle);
      });
      
这里创建了一个`RoundRobinHandle`实例，在该实例的构造函数中通过代码：

      this.server = net.createServer(assert.fail);

      if (fd >= 0)
        this.server.listen({ fd });
      else if (port >= 0)
        this.server.listen(port, address);
      else
        this.server.listen(address);  // UNIX socket path.

      this.server.once('listening', () => {
        this.handle = this.server._handle;
        this.handle.onconnection = (err, handle) => this.distribute(err, handle);
        this.server._handle = null;
        this.server = null;
      });

在父进程中生成了一个server，并且通过注册listen的方法将有心的客户端连接到达时执行的onconnection改成了使用自身的this.distribute函数，这个函数我们先记下因为他是后来父进程给子进程派发任务的重要函数。说回getServer的代码，这里通过`RoundRobinHandle`实例的add方法：

      const done = () => {
        if (this.handle.getsockname) {
          const out = {};
          this.handle.getsockname(out);
          // TODO(bnoordhuis) Check err.
          send(null, { sockname: out }, null);
        } else {
          send(null, null, null);  // UNIX socket.
        }

        this.handoff(worker);  // In case there are connections pending.
      };

      // Still busy binding.
      this.server.once('listening', done);
	

会给子进程的`getServer`以回复。从这里我们可以看到在给子进程的回复中handle一直都是null。那这个所谓的去取得父进程的server是怎么取得的呢？这个地方让我困惑了一下，不过后来看子进程的代码我就明白了，实际上根本不存在什么取得父进程server的句柄，这个地方的注释迷惑了阅读者，从之前子进程的回调中我们可以看到，返回的handle只是决定子进程是用shared方式还是Round-robin的方式来处理父进程派下来的任务。从这个回调函数我们就可以看出，子进程是没有任何获取句柄的操作的，那它是如何处理的呢？我们通过该例子中的rr方法可以看到：

	  const handle = { close, listen, ref: noop, unref: noop };

      if (message.sockname) {
        handle.getsockname = getsockname;  // TCP handles only.
      }

      handles[key] = handle;
      cb(0, handle);

这个函数中生成了一个自带listen和close方法的对象，并传递给了函数`listenOnMasterHandle`，虽然这个名字写的是在父进程的server句柄上监听，实际上我们这个例子中是子进程自建了一个handle，但是如果是udp的情况下这个函数名字还确实就是这么回事，原因在于`SO_REUSEADDR`选项，里面有这样一个解释：
	
	SO_REUSEADDR允许完全相同的地址和端口的重复绑定。但这只用于UDP的多播，不用于TCP。
	
所以，在udp情况同一个地址和端口是可以重复监听的（之前网上看到那个哥们儿说的也没问题，只是一叶障目了），所以可以共享父进程的handle，跟TCP的情况不同。我们继续来看当前这个TCP的情况，在这个情况下`listenOnMasterHandle`会将我们在子进程中自己生成的handle对象传入子进程中通过createServer创建的server的_handle属性中并通过

	server._listen2(address, port, addressType, backlog, fd);
	
做了一个假的监听操作，实际上因为_handle的存在这里只会为之前_handle赋值一个onconnection函数，这个函数的触发则跟父进程中通过真实的客户端连接触发的时机不同，而是通过

	process.on('internalMessage', (message, handle) {
      if (message.act === 'newconn')
        onconnection(message, handle);
      else if (message.act === 'disconnect')
        _disconnect.call(worker, true);
	}
	
中注册的`internalMessage`事件中的对父进程传入的act为newconn的包触发。而父进程中就通过我们刚刚说到的改写了server对象的onconnection函数的distribute函数，这个函数中会调用一个叫handoff的函数，通过代码：

      const message = { act: 'newconn', key: this.key };
      sendHelper(worker.process, message, handle, (reply) => {
        if (reply.accepted)
          handle.close();
        else
          this.distribute(0, handle);  // Worker is shutting down. Send to another.

        this.handoff(worker);
      });

其中send到子进程的handle就是新连接客户端的句柄，Node中父子进程之间的通信最后是通过`src/stream_base.cc`中的`StreamBase::WriteString`函数实现的，从这段代码我们可以看出：

	...
	//当进程间通信时
	uv_handle_t* send_handle = nullptr;

    if (!send_handle_obj.IsEmpty()) {
      HandleWrap* wrap;
      ASSIGN_OR_RETURN_UNWRAP(&wrap, send_handle_obj, UV_EINVAL);
      send_handle = wrap->GetHandle();
      // Reference LibuvStreamWrap instance to prevent it from being garbage
      // collected before `AfterWrite` is called.
      CHECK_EQ(false, req_wrap->persistent().IsEmpty());
      req_wrap_obj->Set(env->handle_string(), send_handle_obj);
    }

    err = DoWrite(
        req_wrap,
        &buf,
        1,
        reinterpret_cast<uv_stream_t*>(send_handle));
        
可以看到，在调用此方式时，如果传入了一个客户端的句柄则通过`Dowrite`方法最后通过辅助数据cmsg_data将客户端句柄的套接字fd传送到子进程中进行处理。看到这里我不禁恍然大悟，原来还是走的是我熟悉的那套网络编程的逻辑啊。

#### 总结

通过上面的一轮分析，我们可以总结出以下两个结论：

1. 创建TCP服务器时，会在父进程中创建一个server并监听目标端口，新连接到达后，会通过ipc的方式将新连接的句柄分配到子进程中然后处理新连接的数据和请求，所以只有主进程会监听目标ip和端口。
2. 创建UDP服务器，会共享在父进程中创建的server的句柄对象，并且在子进程中都会监听到跟对象相同的ip地址和端口上，所以创建n个子进程则会有n+1个进程同时监听到目标ip和端口上。


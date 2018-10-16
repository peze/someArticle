### Node中异步和同步的实现

使用过node的朋友都知道，它最重要的也是最值得称道的就是使用了异步事件驱动的框架libuv，这个框架使得被称为玩具语言的JavaScript也在后端语言中占了一席之地（当然V8的高性能也是功不可没，而且libuv的代码非常优雅，很值得大家的学习。不过libuv整个框架很大，我们不可能只通过一篇文章就能了解到它所有的东西，所以我挑选了node中最简单fs模块同步读和异步读文件的过程讲解来对libuv的一个大概过程有所了解。

#### fs.readSync

`fs.readSync`这个方法我相信没有人会陌生，在node中同步读取文件，不过再很多文章中都不推荐使用这个方法，因为会造成node单线程的阻塞，对于一些比较繁忙的node实例来说是非常不友好的，不过今天我们不讨论这些，只讨论其中的实现。实现我们在node工程的lib目录中找到fs.js可以看到它的代码: 
	
	function(fd, buffer, offset, length, position) {
		if (length === 0) {
			return 0;
		}

		return binding.read(fd, buffer, offset, length, position);
	};
	 
其中直接调用了`binding.read`其中的binding的申明是这样的`binding = process.binding('fs')`，这个自然就是node的builtin_module，所以我们直接找到src/node_flie.cc文件中。

从`node::InitFs`方法中我们可以看到所有的方法申明，其中返回对象中的read方法对应的是`static void Read(const FunctionCallbackInfo<Value>& args)`我们来看一下他的核心代码：

	static void Read(const FunctionCallbackInfo<Value>& args) {
		//获取传入参数，并对参数进行处理
		....
		//将传入的buffer的内存的地址取出并用来存储read出的内容
		char * buf = nullptr;
		Local<Object> buffer_obj = args[1]->ToObject(env->isolate());
		char *buffer_data = Buffer::Data(buffer_obj); 		size_t buffer_length = Buffer::Length(buffer_obj);
		...
		buf = buffer_data + off;

		uv_buf_t uvbuf = uv_buf_init(const_cast<char*>(buf), len);
		//执行read操作
		req = args[5];

		if (req->IsObject()) {
			ASYNC_CALL(read, req, UTF8, fd, &uvbuf, 1, pos);
		} else {
			SYNC_CALL(read, 0, fd, &uvbuf, 1, pos)
			args.GetReturnValue().Set(SYNC_RESULT);
		}
	}
	
从上面的代码，我们可以看出第六个参数是个很关键的参数，如果传入了一个对象则使用异步操作，而我们的`fs.readSync`方法没有传入第六个参数随意使用的是同步操作，并且在操作完成后立即返回结果。

而`SYNC_CALL`这个宏中具体做了什么呢，他主要是调用另外一个宏:

	#define SYNC_CALL(func, path, ...)  \
	SYNC_DEST_CALL(func, path, nullptr, __VA_ARGS__) \
其中__VA_ARGS__表示的是除了func和path外其他传入宏的参数，接下来我们来看一下SYNC_DEST_CALL宏:
	
	#define SYNC_DEST_CALL(func, path, dest, ...)                                 \
		fs_req_wrap req_wrap;                                                     \
		env->PrintSyncTrace();                                                    \
		int err = uv_fs_ ## func(env->event_loop(),                               \
		                     &req_wrap.req,                                       \
		                     __VA_ARGS__,                                         \
		                     nullptr);                                            \
		if (err < 0) {                                                            \
			return env->ThrowUVException(err, #func, nullptr, path, dest);        			\
		}                                                                         \
		

其中在宏命令中的 `##`标记是连接符的意思，所以这里其实就是调用uv_fs_read方法，而`env->PrintSyncTrace()`是为了在node打开`--trace-sync-io`时用来追踪代码中何处使用了同步io时使用，可以通过这个方法打出代码中调用同步io的位置，所以当你的代码经常发生阻塞的时候你可以通过这个来调优你的代码（当然阻塞的原因未必是同步io造成的）。uv_fs_read方法是libuv是来读取文件的调用，我们找到这个方法的位置，就在deps/uv/src/unix/fs.c中:

	int uv_fs_read(uv_loop_t* loop, uv_fs_t* req,
	           uv_file file,
	           const uv_buf_t bufs[],
	           unsigned int nbufs,
	           int64_t off,
	           uv_fs_cb cb) {
		INIT(READ);

		if (bufs == NULL || nbufs == 0)
		return -EINVAL;

		req->file = file;

		req->nbufs = nbufs;
		req->bufs = req->bufsml;
		if (nbufs > ARRAY_SIZE(req->bufsml))
			req->bufs = uv__malloc(nbufs * sizeof(*bufs));

		if (req->bufs == NULL) {
		if (cb != NULL)
			uv__req_unregister(loop, req);
			return -ENOMEM;
		}

		memcpy(req->bufs, bufs, nbufs * sizeof(*bufs));

		req->off = off;
		POST;
	}
	
首先我们来看宏调用`INIT(READ)`:
	
	#define INIT(subtype)                            	 \
		do {                                             \
			if (req == NULL)                             \
				return -EINVAL;                          \
			req->type = UV_FS;                           \
			if (cb != NULL)                              \
				uv__req_init(loop, req, UV_FS);          \
			req->fs_type = UV_FS_ ## subtype;            \
			req->result = 0;                             \
			req->ptr = NULL;                             \
			req->loop = loop;                            \
			req->path = NULL;                            \
			req->new_path = NULL;                        \
			req->cb = cb;                                \
		}                                                \
		while (0)
	
这是一个很明显的初始化操作，这里主要说两个最重要地方，首先是将req的loop指向node的event_loop，其次是指定了fs_type喂`UV_FS_READ`这个是一个重要的标志，为后面的工作做识别。做了这些操作以后回到`uv_fs_read`方法来，我们可以看到在POST宏调用之前都是一些对参数的处理工作，这个没什么可讲的，我们主要来看看POST宏:
	
	#define POST                                                                  \
		do {                                                                      \
			if (cb != NULL) {                                                     \
				uv__work_submit(loop, &req->work_req, uv__fs_work, uv__fs_done);  \
				return 0;                                                         \
			}                                                                     \
			else {                                                                \
				uv__fs_work(&req->work_req);                                      \
				return req->result;                                               \
			}                                                                     \
		}                                                                         \
		while (0)
		
从上面的代码我们可以看到在有cb的时候调用的是uv__work_submit，这就是异步的情况下调用，等会儿我们再讲。现在我们先说uv__fs_work
的方法：

	static void uv__fs_work(struct uv__work* w) {
		int retry_on_eintr;
		uv_fs_t* req;
		ssize_t r;

		req = container_of(w, uv_fs_t, work_req);
		retry_on_eintr = !(req->fs_type == UV_FS_CLOSE);

		do {
			errno = 0;

			#define X(type, action)                               \
				case UV_FS_ ## type:                              \
				r = action;                                       \
				break;

				switch (req->fs_type) {
					...
					X(WRITE, uv__fs_buf_iter(req, uv__fs_write));
					X(OPEN, uv__fs_open(req));
					X(READ, uv__fs_buf_iter(req, uv__fs_read));
					...
				}
			#undef X
		} while (r == -1 && errno == EINTR && retry_on_eintr);

		if (r == -1)
			req->result = -errno;
		else
			req->result = r;

		if (r == 0 && (req->fs_type == UV_FS_STAT ||
		             req->fs_type == UV_FS_FSTAT ||
		             req->fs_type == UV_FS_LSTAT)) {
			req->ptr = &req->statbuf;
		}
	}
	
这个方法因为是fs文件共用的方法，所以在其中会根据不同的类型的来执行不同的方法，刚刚我们看到了在初始化req的时候给了它UV_FS_READ的type，所以会执行方法`uv__fs_buf_iter(req, uv__fs_read)`,uv__fs_buf_iter方法中主要是调用了传入的第二个参数`uv__fs_read`函数，这里的代码就不贴了很简单，就是普通的read(还有readv和pread)操作，不过其中有个点就是这段代码:

	#if defined(_AIX)
		struct stat buf;
		if(fstat(req->file, &buf))
			return -1;
		if(S_ISDIR(buf.st_mode)) {
			errno = EISDIR;
			return -1;
		}
	#endif
	
这段代码很好地解释了node文档中关于`fs.readFileSync`的这一段

	Note: Similar to fs.readFile(), when the path is a directory, the behavior of fs.readFileSync() is platform-specific.

	// macOS, Linux, and Windows
	fs.readFileSync('<directory>');
	// => [Error: EISDIR: illegal operation on a directory, read <directory>]

	//  FreeBSD
	fs.readFileSync('<directory>'); // => null, <data>
	
在`uv__fs_read`成功读取文件后,req->bufs中就已经有了所需的内容了，从node_file.cc的`static void Read(const FunctionCallbackInfo<Value>& args)`方法中我们可以知道req->bufs的内存所指向的则是`binding.read(fd, buffer, offset, length, position)`传入的buffer内存段。这个时候就已经得到了想要读取的内容了。而我们平时经常使用的`fs.readFileSync`则是先打开文件得到其fd，并生成一段buffer然后调用`fs.readSync`，是生成的buffer中取得文件内容再返回，简化了很多操作，所以更受到大家的青睐。到这里我们的同步读取就已经结束了，算是很简单，因为read这些操作都是阻塞性的操作，所以对于单线程的node进程来说确实容易遇到性能瓶颈，下面我们来说一下node的异步读取`fs.read`函数。

#### fs.read

异步的操作远比同步要复杂很多，我们来一步步的了解。首先我们先来看
ocess.nextTick(function() {
				callback && callback(null, 0, buffer);
			});
		}

		function wrapper(err, bytesRead) {
			// Retain a reference to buffer so that it can't be GC'ed too soon.
			callback && callback(err, bytesRead || 0, buffer);
		}

		var req = new FSReqWrap();
		req.oncomplete = wrapper;

		binding.read(fd, buffer, offset, length, position, req);
	};
	
从刚刚同步的分析中，我们知道当`bingd.read`传入第六个参数的时候则会异步执行read操作，这里就传入了第六个参数req, `req = new FSReqWrap();`req是`FSReqWrap = binding.FSReqWrap`的实例，所以我们从`node::InitFs`中可以看到如下代码:

	Local<FunctionTemplate> fst = FunctionTemplate::New(env->isolate(), NewFSReqWrap);
	fst->InstanceTemplate()->SetInternalFieldCount(1);
	AsyncWrap::AddWrapMethods(env, fst);
	Local<String> wrapString =
	FIXED_ONE_BYTE_STRING(env->isolate(), "FSReqWrap");
	fst->SetClassName(wrapString);
	target->Set(wrapString, fst->GetFunction());

上面的代码使用v8提供的API生成FSReqWrap的构造函数而`void NewFSReqWrap(const FunctionCallbackInfo<Value>& args)`就会说起构造函数的内容。这个函数主要的主要工作只有一个`object->SetAlignedPointerInInternalField(0, nullptr);`，不过这个只跟C++对象的嵌入有关。从之前我们讨论过的`static void Read(const FunctionCallbackInfo<Value>& args)`方法中聊到过，当传入req对象的时候回调用宏命令ASYNC_CALL，这个宏命令跟之前的SYNC_CALL一样的调用，通过`ASYNC_DEST_CALL(func, req, nullptr, encoding, __VA_ARGS__)`去调用真正的逻辑，所以我们直接来看`ASYNC_DEST_CALL`的代码：

	#define ASYNC_DEST_CALL(func, request, dest, encoding, ...)                   \
		Environment* env = Environment::GetCurrent(args);                         \
		CHECK(request->IsObject());                                               \
		FSReqWrap* req_wrap = FSReqWrap::New(env, request.As<Object>(),           \
		                                   #func, dest, encoding);                \
		int err = uv_fs_ ## func(env->event_loop(),                               \
		                       req_wrap->req(),                                   \
		                       __VA_ARGS__,                                       \
		                       After);                                            \
		req_wrap->Dispatched();                                                   \
		if (err < 0) {                                                            \
			uv_fs_t* uv_req = req_wrap->req();                                    \
			uv_req->result = err;                                                 \
			uv_req->path = nullptr;                                               \
			After(uv_req);                                                        \
			req_wrap = nullptr;                                                   \
		} else {                                                                  \
			args.GetReturnValue().Set(req_wrap->persistent());                    \
		}
		
上面的代码我们可以看到通过FSReqWrap::New来生成了req_wrap，这个方法的执行是node生成对象的一个基本逻辑，所以我们着重说一下，首先我们来看一下`FSReqWrap::New`的代码:

	const bool copy = (data != nullptr && ownership == COPY);
	const size_t size = copy ? 1 + strlen(data) : 0;
	FSReqWrap* that;
	char* const storage = new char[sizeof(*that) + size];
	that = new(storage) FSReqWrap(env, req, syscall, data, encoding);
	if (copy)
		that->data_ = static_cast<char*>(memcpy(that->inline_data(), data, size));
	return that;
	
这段代码我们主要了解一下`new(storage) FSReqWrap(env, req, syscall, data, encoding);`，首先我们通过一张图来了解一下FSReqWrap的继承关系：

![image1](http://blog.magicare.me/content/images/2018/10/1539423697038.jpg)
	
上图中我们给出了一些关键对象的关键属性和方法，所以我们可以看出FSReqWrap各个继承对象的主要作用：

1.继承ReqWrap对象的关键属性uv_fs_t，和关键方法`ReqWrap<T>::Dispatched`，使用该方法中的`req_.data = this;`在libuv的方法中传递自身。

2.继承AsyncWrap中的MakeCallback，这个函数会执行我们传入的异步读取完成后的回调，在这个例子中就是使用js中通过`req.oncomplete = wrapper;`传入的wrapper函数。

3.继承BaseObject对象中的关键属性`Persistent<Object> persistent_handle_`和`Environment* env_`，前者是v8中的持久化js对象，和Local<Object>的关系可以参见v8官方的解释：

	Local handles are held on a stack and are deleted when the appropriate destructor is called. These handles' lifetime is determined by a handle scope, which is often created at the beginning of a function call. When the handle scope is deleted, the garbage collector is free to deallocate those objects previously referenced by handles in the handle scope, provided they are no longer accessible from JavaScript or other handles.
	
	Persistent handles provide a reference to a heap-allocated JavaScript Object, just like a local handle. There are two flavors, which differ in the lifetime management of the reference they handle. Use a persistent handle when you need to keep a reference to an object for more than one function call, or when handle lifetimes do not correspond to C++ scopes. 
	
大概的意思就是，Local<Object>会随着在栈上分配的scope析构而被GC清理掉，但是Persistent<Object>不会。有点类似栈上分配的内存和堆上分配内存的关系，想要在超过一个function中使用就要使用Persistent的v8对象，而后者是node的执行环境，几乎囊括了node执行中所需要的一切方法和属性（这一块非常大，涉及的也很多，实在很难一两句讲清楚，跟本文讨论内容无直接联系只能略过）。

最后，在FSReqWrap的构造函数中通过`Wrap(object(), this)`将我们上面提到的`Persistent<Object> persistent_handle_`持久化js对象和FSReqWrap的C++对象关联起来，这是node中最常用的方式（也是ebmed开发中最常用的技巧）。我们回到宏`ASYNC_DEST_CALL`中来，现在知道通过方法`FSReqWrap::New`方法使得FSReqWrap对象实例和刚刚在js中new的req对象连接了起来，也使libuv的uv_fs_t和其实例联系了起来。这个时候就跟前面一样开始调用`uv_fs_read`，这次在最后一个参数cb中传入了函数`void After(uv_fs_t *req)`作为回调函数，从之前同步讨论中我们就说过传入回调函数后情况的不同，首先是`INIT`宏中在会多一步操作，通过`uv__req_init`中的`QUEUE_INSERT_TAIL(&(loop)->active_reqs, &(req)->active_queue);`宏方法将req放入loop的acitve_reqs的循环链表中（libuv的循环链表实现非常的有意思，有兴趣的朋友可以参考文章:[libuv queue的实现](https://www.jianshu.com/p/6373de1e117d))。而在`POST`中有回调的函数的情况是直接通过`uv__work_submit(loop, &req->work_req, uv__fs_work, uv__fs_done)`调用来完成任务，我们来看一下`uv__work_submit`函数的代码，这个方法在deps/uv/src/threadpool中：

	uv_once(&once, init_once);
	w->loop = loop;
	w->work = work;
	w->done = done;
	post(&w->wq);
	
该方法首先通过uv_once在第一次调用该方法是启动几个工作线程，这些线程主要执行`static void worker(void* arg)`方法：
	
	for (;;) {
		uv_mutex_lock(&mutex);

		while (QUEUE_EMPTY(&wq)) {
			idle_threads += 1;
			uv_cond_wait(&cond, &mutex);
			idle_threads -= 1;
		}

		q = QUEUE_HEAD(&wq);

		if (q == &exit_message)
			uv_cond_signal(&cond);
		else {
			QUEUE_REMOVE(q);
			QUEUE_INIT(q);  
		}

		uv_mutex_unlock(&mutex);

		if (q == &exit_message)
			break;

		w = QUEUE_DATA(q, struct uv__work, wq);
		w->work(w);

		uv_mutex_lock(&w->loop->wq_mutex);
		w->work = NULL;  
		QUEUE_INSERT_TAIL(&w->loop->wq, &w->wq);
		uv_async_send(&w->loop->wq_async);
		uv_mutex_unlock(&w->loop->wq_mutex);
	}
	
其中wq是一个循环链表的队列，记录了所有注册的任务，当没有任务时会通过`uv_cond_wait`使该线程阻塞，而在有任务的时候会在队列中取出该任务再通过`w->work(w)`执行其任务，在执行完成后会将任务注册在`loop->wq`的队列中再通过`uv_async_send(&w->loop->wq_async)`通知主线程从`loop->wq`的队列取出该任务并执行其回调。

再回到`uv__work_submit`通过`work`方法我们就知道它接下来的工作是做什么了，注册work函数也就是传入`uv__fs_work`函数，这个函数我们之前就介绍过了，这里就不多做解释了，只是在异步中是通过worker线程来完成的，不会阻塞主线程。而第二个函数则是注册完成后主线执行的回调，也就是`uv__fs_done`:
	
	req = container_of(w, uv_fs_t, work_req);
	uv__req_unregister(req->loop, req);

	if (status == -ECANCELED) {
		assert(req->result == 0);
		req->result = -ECANCELED;
	}

	req->cb(req);
	
从中我们可以看到，这个函数会将该任务的req从loop的acitve_reqs去去掉，然后执行传入uv_fs_read中的回调函数。而最后的post中主要是将当前任务注册到wq的列表中，并使用条件变量的`uv_cond_signal`函数触发`uv_cond_wait`中阻塞的函数运作起来，接着worker进程就能执行我们刚刚说的过程了。

上面我们讲解了大概的过程，从这个过程中就能明白异步的读操作是如何执行的，通过使用wokrer线程来做实际的读操作，而主线程则是在worker线程完成操作后，执行回调。不过现在回过头来我们看看，在worker线程以后是如何通知主线程呢？刚刚我们说到了是通过`uv_async_send(&w->loop->wq_async)`的调用通知的，这里我们来看看他具体是如何做的。首先我们要回到loop的初始化处，函数`uv_loop_init`中，在这个函数中有这样一个调用： `uv_async_init(loop, &loop->wq_async, uv__work_done);`。这个调用会生成一个管道，并通过以下语句：
	
	uv__io_init(&loop->async_io_watcher, uv__async_io, pipefd[0]);
	uv__io_start(loop, &loop->async_io_watcher, POLLIN);
	loop->async_wfd = pipefd[1];
	
实现当有数据往pipefd[1]中写时，主线会在读取数据后执行`uv__async_io`的调用，在`uv__async_io`中最重要的工作就是执行其async_cb，而在loop初始化的时候注册的async_cb是函数`uv__work_done`:
	
	//取数据的操作
	...

	while (!QUEUE_EMPTY(&wq)) {
		q = QUEUE_HEAD(&wq);
		QUEUE_REMOVE(q);

		w = container_of(q, struct uv__work, wq);
		err = (w->work == uv__cancelled) ? UV_ECANCELED : 0;
		w->done(w, err);
	}
	
这里我们可以看到，会从`loop->wq`队列中取出放入其中所有任务，并通过`w->done(w, err)`执行其回调，而刚刚在worker线程中的调用`uv_async_send(&w->loop->wq_async)`即是通过往`loop->async_wfd`，即上面提到的pipefd[1]写一个字节来触发整个过程。到这里最开始`uv_fs_read`中注册的函数`uv__fs_done`就可以执行了，而这个函数的主要任务即是调用传入`uv_fs_read`的cb参数，即`void After(uv_fs_t *req)`函数，这个函数处理的情况比较多，就不贴代码了，唯一要讲的就是他的第一句

	FSReqWrap* req_wrap = static_cast<FSReqWrap*>(req->data);

这里就回到了我前面所说的通过req->data将FSReqWrap的对象实例串联起来，到这里就能顺利的通过这个实例得到之前初始化的js对象，并执行它的oncomplete函数了。回到js的代码中我们可以看到这个函数执行的操作就是调用我们传入的callback的函数:
	
	callback && callback(err, bytesRead || 0, buffer);
	
至此，fs.read整个异步操作就已经完成了，至于fs.readFile这个操作放在异步中就复杂了许多，先异步打开文件，再通过回调中注册异步任务取得文件的stat，最后通过回调去读取文件，而且如果文件太大不能一次读完（一次最多读8*1024的字节），会不断的回调继续读取文件，直到读完才异步关闭文件，并且通过异步关闭文件的回调执行传入的回调函数。可见为了我们平时开发中的方便，node的开发者还是付出了很多的努力的。


####总结


在了解了node对于文件读取的同步和异步实现后，我们就能看出libuv的精妙之处了。特别是异步时通过子线程处理任务，再用管道通知主线执行回调的方式，真的是为node这样的单线程语言量身定做，当然可能也有同学有疑问，主线是如何读取管道值的呢？这又是一个很大的问题，我们只能以后的文章再来解释了。这篇文章就先到此为止了，希望通过该文能帮助大家对node背后的逻辑会多一点了解。
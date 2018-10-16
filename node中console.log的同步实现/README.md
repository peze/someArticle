### Node中console.log的同步实现

console.log相信使用过js的朋友都不会陌生，对于我这种前端转过来的node开发者，用起这个函数更是毫不手软，使用它把需要的信息打印到标准输出，觉得就是1+1=2那么正常，但是有天在网上看到一个问题console.log到底是异步还是同步？我觉得很诧异，这还是个问题么？当然是同步啦。但是问题的答案出乎我的意料，上面告诉我是要分情况的，根据process.stdout的情况可能会出现异步的情况。我当时眉头一皱，才发现问题确实不是我想的那么简单，于是在Node的文档中发现了这一段提示：
	
	Writes may be synchronous depending on what the stream is connected to and whether the system is Windows or POSIX:
		Files: synchronous on Windows and POSIX
		TTYs (Terminals): asynchronous on Windows, synchronous on POSIX
		Pipes (and sockets): synchronous on Windows, asynchronous on POSIX

当我发现自己对这个知识存在盲区后，赶紧深入内核去看看到底是啥情况，我选择了最常用的POSIX上的TTYs来深入理解。

#### 从console.log出发

首先我在node源文件中从lib/console.js找到了console.log的代码：

	Console.prototype.log = function log(...args) {
	  write(this._ignoreErrors,
	        this._stdout,
	        util.format.apply(null, args),
	        this._stdoutErrorHandler,
	        this[kGroupIndent]);
	};

这个中间包含了一些格式化字符串之类的东西，不过其核心还是很明显的就是write函数中的
	
	stream.write(string, errorhandler);

而stream就是`this._stdout`，从代码:

	module.exports = new Console(process.stdout, process.stderr);
	module.exports.Console = Console;
	
中我们又可以知道，这个`this._stout`就是`process.stdout`，那上面的问题也解释的通了，所以console.log到底是同步输出还是异步输出还真得看情况了。

#### process.stdout的实现

现在我们将目光转向`process.stdout`,对于这个属性的定义在lib/internal/process/stdiso.js中，通过分析该文件，我们可以发现stdout的stream是这样定义的：
	
	const tty_wrap = process.binding('tty_wrap');
	switch (tty_wrap.guessHandleType(fd)) {
		case 'TTY':
			var tty = require('tty');
			stream = new tty.WriteStream(fd);
			stream._type = 'tty';
		break;

		case 'FILE':
			var fs = require('internal/fs');
			stream = new fs.SyncWriteStream(fd, { autoClose: false });
			stream._type = 'fs';
		break;

		case 'PIPE':
		case 'TCP':
			var net = require('net');
			stream = new net.Socket({
				fd: fd,
				readable: false,
				writable: true
			});
			stream._type = 'pipe';
		break;

		default:
		// Probably an error on in uv_guess_handle()
		throw new errors.Error('ERR_UNKNOWN_STREAM_TYPE');
	}

	// For supporting legacy API we put the FD here.
	stream.fd = fd;

	stream._isStdio = true;
	
从上面就可以看出文档中的提示，file状态使用了`fs.SyncWriteStream`自然是同步的，而PIPE是用`net.Socket`实现的，在Posix标准的机器上自然是异步的。而让我最困惑的是TTY的实现方式，其中的`tty.WriteStream`在lib/tty.js中是这样实现的：
	
	function WriteStream(fd) {
		...

		net.Socket.call(this, {
			handle: new TTY(fd, false),
			readable: false,
			writable: true
		});

		this._handle.setBlocking(true);

		...
	}
	inherits(WriteStream, net.Socket);
	
可以看到TTY时的stream是继承`net.Socket`的，连new的时候构造函数都是直接调用它的构造函数，只是handle是TTY对象的。刚刚才说过`net.Socket`不应该是异步的吗？到这儿来咋就成异步的了呢？这个时候我就产生了一丝不解，想知道它是如何使TTY方式下的stdout变成同步的。于是翻起了源码，既然tty的stream是继承`net.Socket`所以，而`net.Socket`对象是一个标准的node流对象，他直接继承自`stream.Duplex`这个双全工的流对象，所以我们可以直接到lib/_stream_writable.js中找到方法`Writable.prototype.write`，通过分析它的代码我们可以知道实际上调用的是`Socket.prototype._writeGeneric`函数，而在这个函数中会根据不同的字符类型选择调用不同的stream方法：
	
	switch (encoding) {
		case 'latin1':
		case 'binary':
			return handle.writeLatin1String(req, data);

		case 'buffer':
			return handle.writeBuffer(req, data);

		case 'utf8':
		case 'utf-8':
			return handle.writeUtf8String(req, data);

		case 'ascii':
			return handle.writeAsciiString(req, data);

		case 'ucs2':
		case 'ucs-2':
		case 'utf16le':
		case 'utf-16le':
			return handle.writeUcs2String(req, data);

		default:
			return handle.writeBuffer(req, Buffer.from(data, encoding));
	}	

	
而这个例子中的handle为TTY的实例，TTY是通过`process.bingding`得到的，所以这些方法是`NODE_BUILTIN_MODULE`的方法。上面的这些方法都是调用src/stream_base.cc中的模板函数

	template <enum encoding enc>
	int StreamBase::WriteString(const FunctionCallbackInfo<Value>& args)
	
从这个函数中我们可以看到，虽然编码不同会造成在生成`stack_storage`值时所用的处理方式不同，但是最后都是通过

	err = DoWrite(
        req_wrap,
        &buf,
        1,
        reinterpret_cast<uv_stream_t*>(send_handle));
        
操作来完成写操作的，`DoWrite`是个纯虚函数，这个函数的实际定义实在TTYWrap的基类，streamBase的派生类中定义的，在文件src/stream_wrap.cc中定义，在其中使用了libuv的方法`uv_write`来执行io真正的写操作。紧接着我又把目光转向了文件deps/uv/src/unix/stream.c中的这个方法，其中调用了`uv_write2`，这是libuv中一个很经典的异步方法，但是也有特例从其中执行写操作的实际函数`uv__write`中我们可以看到，如果当前的这个流设置了`UV_STREAM_BLOCKING`标记，则会一直同步写完，并不会出现异步操作。那我们的TTY是在哪儿设置的这个标记？我们可以回到lib/tty.js中的这句话：
	
	net.Socket.call(this, {
    	handle: new TTY(fd, false),
    	readable: false,
    	writable: true
    });
    
这里TTY对象刚刚我们说了是node的内部对象，所以这里实际会调用的是src/tty_wrap.cc中的`void TTYWrap::New(const FunctionCallbackInfo<Value>& args)`函数，其中通过：
	
	TTYWrap* wrap = new TTYWrap(env, args.This(), fd, args[1]->IsTrue(), &err);
	
生成TTYWrap的实例，而TTYWrap对象的构造函数中通过：
	
	uv_tty_init(env->event_loop(), &handle_, fd, readable);
	
初始化tty的libuv stream handle,从`uv_tty_init`的代码中可以知道，当readable参数为false时就会给handle设置`UV_STREAM_BLOCKING`标记，而readable参数是通过`new TTY(fd, false)`第二个参数传入的，刚好是false，所以process.stdout自然是同步的咯。

#### 总结

以前一直觉得自己对node已经很熟悉了，发现是`net.Socket`的流操作时，虽然有点困惑，但也是觉得可能handle是TTY的实例，写操作会不一样，但是在源码中一步步探索，最后发现还是通过libuv的uv_write2的时候，变得异常困惑，因为之前一直觉得它就是通过异步来完成写操作的，而忽略了设置`UV_STREAM_BLOCKING`的情况，最后是在通过在uv_tty_init和其他的流初始化中比较，发现了tty中出现了设置`UV_STREAM_BLOCKING`的情况，再回过头去找，才发现了设置该标志的写操作是同步的情况。通过这件事还是明白了，很多东西不能想当然，得自己多探索了解才能在技术上面沉淀的更多，希望我的这篇文章也同时能帮助到大家。
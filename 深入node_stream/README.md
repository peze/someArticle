### 深入node stream

要说node最令人印象深刻的模块，第一肯定是events，而第二肯定就是stream模块了，今天这篇文章就来跟大家聊聊stream的实现，主要是以stream_readable为例来讲解，并以fs模块中的createReadStream为例来说明node内部是如何使用stream的。


#### Stream.Readable

从lib/stream.js中我们可以看到node的stream主要分为了四种，读流，写流以及读写兼具的全双工流，还有可以修改和变换数据的 Transform流 ，不过读流和写流的实现逻辑和思路在大架构上是类似的，至于双全工流我们可以通过文件lib/_stream_duplex.js看到他其实就是通过:

	function Duplex(options) {
		...
		Readable.call(this, options);
		Writable.call(this, options);
		...
	}
	
使函数拥有了读流和写流所有的方法和属性而成，而Transform流其实是 Duplex流重写了_read和_write方法的派生流，所以本文通过介绍读流的实现方式来理解整个stream的实现方式。

首先，可以从lib/stream.js中发现stream的真实位置来源于lib/internal/streams/legacy.js，从这个文件中我们可以看到stream也继承于events，并且这里还提供了一个pipe方法，这个方法我们后面讲例子的时候再讲。而从lib/_stream_readable.js文件中可以看出，虽然提供了能组成整套流程的方法，但是最核心的`Readable.prototype._read`中却是这样：
	
	this.emit('error', new Error('_read() is not implemented'));
	
会直接爆出错误，为什么呢？注释中已经给名了原因，该函数是一个抽象方法，是不能被直接调用的，只能通过其他实际的读流来继承该类再重写这个方法后使用，可以将Stream.Readable理解为一个抽象类，所以我们直接通过一个fs中的使用用例来说明。

#### fs.createReadStream

下面我们直接从lib/fs.js的`fs.createReadStream`方法来看Stream.Readable，这个方法中返回了一个ReadStream实例，而ReadStream是通过`util.inherits(ReadStream, Readable);`关联上了Stream.Readable，在 `new ReadStream`的时候会通过`new ReadableState`创建大量控制流操作的属性，后面我们都会讲到。现在让我们来看一个最简单的例子：
	
	var stream = fs.createReadStream('sample.txt');
	stream.on('data', (chunk) => {
    	console.log('读取文件数据:', chunk);
	});
	
这个例子中，在创建了读流以后，就只需要注册events事件，在有数据的时候来触发就行了，这是怎么实现的呢？原来在`Stream.Readable`中重写了events的on方法：
	
	Readable.prototype.on = function(ev, fn) {
		const res = Stream.prototype.on.call(this, ev, fn);

		if (ev === 'data') {
			if (this._readableState.flowing !== false)
				this.resume();
		} else if (ev === 'readable') {
			const state = this._readableState;
			if (!state.endEmitted && !state.readableListening) {
				state.readableListening = state.needReadable = true;
				state.emittedReadable = false;
				if (!state.reading) {
					process.nextTick(nReadingNextTick, this);
				} else if (state.length) {
					emitReadable(this);
				}
			}
		}

		return res;
	};
	
其中当注册data和readable事件的时候，我们先以例子中data事件举例往下分析，这里调用了`ReadStream.prototype.resume`函数，这个函数会在最后通过`process.nextTick`调用函数`resume_`,所以我们可以直接来看看这个函数的代码:
	
	if (!state.reading) {
		debug('resume read 0');
		stream.read(0);
	}

	state.resumeScheduled = false;
	state.awaitDrain = 0;
	stream.emit('resume');
	flow(stream);
	if (state.flowing && !state.reading)
		stream.read(0);

其中的state就是`ReadableState`的实例，当异步读取未完成时reading属性为true，不过我们的代码刚开始为初始值false所以这里会直接调用`stream.read(0);`。现在让我们来看一下`Readable.prototype.read`的代码，这个方法在这次调用时，会直接执行以下的代码：

	....//忽略做判断以及不执行的代码
	state.reading = true;
	state.sync = true;
	if (state.length === 0)
	  state.needReadable = true;
	this._read(state.highWaterMark);
	state.sync = false;
	...//忽略做判断以及不执行的代码
	if (ret === null) {
    	state.needReadable = true;
    	n = 0;
	} 
	
这里主要就是执行重写了的`ReadStream.prototype._read`，这里我们可以看到这个方法第一句代码：

	if (typeof this.fd !== 'number') {
		return this.once('open', function() {
			this._read(n);
		});
	}
	
这是为了保证文件已经被打开了，让我们回到`ReadStream`构造函数中，这里有这样一段代码：
	
	if (typeof this.fd !== 'number')
    	this.open();
    
当我们只传了文件路径的时候，会直接调用`ReadStream.prototype.open`这个函数实际就是使用了`fs.open`打开文件路径，然后在成功回调后通过：

	self.emit('open', fd);
    // start the flow of data.
    self.read();
    
触发open事件，并调用read方法。了解了这个以后，我们接着回来看_read方法：

	//分配buffer，校正读取长度
	...
	//已经不能继续读取了
	if (toRead <= 0)
    	return this.push(null);
	fs.read(this.fd, pool, pool.used, toRead, this.pos, (err, bytesRead) => {
	//错误处理
		if(err){
		...
		}else{
			var b = null;
			if (bytesRead > 0) {
				this.bytesRead += bytesRead;
				b = thisPool.slice(start, start + bytesRead);
			}
    		this.push(b);
    	}
    }
    //记录本次读取后的文件的偏移
    if (this.pos !== undefined)
    	this.pos += toRead;
    

在第一次通过_read成功读取了数据后会调用`Readable.prototype.push`，这个方法调用`addChunk`方法，在这个方法中有这样一段代码:
	
	if (state.flowing && state.length === 0 && !state.sync) {
		stream.emit('data', chunk);
    	stream.read(0);
	}
	
`state.flowing`在之前说的`ReadStream.prototype.resume`中已经设置为true了，而`state.length`为初始值即为0，因为我们是异步调用所以`state.sync`也为false，这个if语句成立后会直接触发data事件，并将读取的数据传递给回调方法。然后通过`stream.read(0);`不断的重复读取，直到读取不到数据时，在_read的方法中通过`this.push(null);`调用方法`onEofChunk`将读操作停止掉。而如果以下面的例子会发生什么不同的情况呢？

	

	var stream = fs.createReadStream('sample.txt');
	stream.on('readable', () => {
    	console.log('文件数据可读');
	});

	
只注册了readable事件的情况呢？我们回过头去看之前`Readable.prototype.on`的方法中对readable事件的处理，通过分析可以了解到会调用`process.nextTick`方法执行函数`nReadingNextTick`，这个函数只有一句话`self.read(0);`，到了这里读取的逻辑跟之前的逻辑就基本一样了，但是在`Readable.prototype.push`中发生了变化，因为`state.flowing`为false，所以就会执行代码：

	state.length += state.objectMode ? 1 : chunk.length;
    if (addToFront)
      state.buffer.unshift(chunk);
    else
      state.buffer.push(chunk);

    if (state.needReadable)
      emitReadable(stream);
      
会将读取到的buffer存到ReadableState的buffer属性中，并通过`emitReadable`触发readble事件的回调。在执行完这些之后，函数也会通过`process.nextTick`注册tick任务，执行如下代码：
	
	var len = state.length;
	while (!state.reading && !state.flowing && !state.ended &&
         state.length < state.highWaterMark) {
    	stream.read(0);
    	if (len === state.length)
      		break;
    	else
      		len = state.length;
	}
	
可以看到当`state.flowing`为true的时候并不会执行，所以在上面我们并没有介绍这个方法，这个方法会通过不断触发read一直执行到state中的buffer缓存区缓存的数据达到highWaterMark所设置的阈值或没有读到新数据为止。那假如在这个时候，我们再注册data事件会怎么样呢?这个时候就会触发我们之前说的`resume_`函数，而在其中之前说到的最开头的调用`stream.read(0)`因为`ReadableState.length`超过了阈值，所以并不会执行_read方法，而是直接跳过，但是在后面的`flow(stream)`中通过`flow(stream);`调用执行代码`while (state.flowing && stream.read() !== null);`，而这里的`stream.read()`会执行以下语句：

	n = parseInt(n, 10);
	...
	n = howMuchToRead(n, state);
	...
	if (n > 0)
    	ret = fromList(n, state);
    ...
    if (ret !== null)
    	this.emit('data', ret);
    return ret;
	
其中因为read调用没传入任何参数，所以`parseInt`的结果为NaN，从`howMuchToRead`中可以看到：

	if (n !== n) {
    	if (state.flowing && state.length)
      		return state.buffer.head.data.length;
    	else
      		return state.length;
	}
	
会返回之前存储在state中的buffer的size，然后通过`fromList(n, state)`从state存储的buffer中取出数据返回，并触发data事件的回调。不过一般而言，如果选择使用注册readable事件，则是选择使用手动的方式来读取内容了。比如：

	var stream = fs.createReadStream('sample.txt',{encoding:'utf8'});
	stream.on('readable', () => {
    	console.log('文件数据为：' + stream.read() );
	});
	



#### 总结

从上面的描述大家可以了解到流中因为属性多，判断多，不同的实现有不同的触发机制所以显得比较混乱，希望通过上文的讲解能让大家对node的stream的设计思路有所了解，本文篇幅有限所以其他比如写流的实现和pipe的实现大家就可以自己去研究一下了，如果有兴趣还能模仿fs的流生成过程，用node的stream派生出自己的stream来做一些尝试，这里给大家一个例子参考：

	var stream = require('stream');
	var util = require('util');
	util.inherits(Coustomer, stream.Readable);
	function Coustomer(options) {
	    stream.Readable.call(this, options);
	    this._str = "";
	}
	Coustomer.prototype._read = function() {
	    if(this._str.length < 10){
	        this._str += "a";
	        this.push(this._str);
	    }else{
	        this.push(null);
	    }
	};
	var coustoomer = new Coustomer();

	coustoomer.on('data', function(data){
	    console.log("读到数据: " + data.toString());//no maybe
	});
	coustoomer.on('end', function(data){
	    console.log("结束");
	});
	

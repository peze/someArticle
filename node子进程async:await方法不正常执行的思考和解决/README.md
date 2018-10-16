###Node子进程async/await方法不正常执行的思考和解决


前段时间，我做了一个node模块[node-multi-worker
](https://github.com/peze/node-multi-worker)，希望通过这个模块让node能够脱离单线程的限制，具体的使用可以看一下上面的链接。其思路就是注册任务后，分出子进程，然后在主进程需要执行任务时，向reactor子进程发送命令，而reactor收到命令后分配到worker子进程在执行完成后返回结果到主进程。这篇文章主要是为了跟大家分享一下我在开发过程中，遇到的一个问题，如何解决以及对相关知识的一个挖掘。


####不执行的async/await

在第一次完成了该工程后，我做了一些简单的测试，比如在子进程执行的方法中做一些加减乘除或者字符运算，当然都是没问题的。而对于一些异步的情况，我通过bluebird的处理也能够处理，于是我开始尝试起了aysnc/await的情况，结果发现这个的执行只要遇到await，await后面的语句能够执行，但是在下面的语句就再也不能执行了。这个情况顿时让我摸不着了头脑，我一度以为是v8内核中对于这种子进程的情况不支持（确实v8对你fork出子进程的支持是有问题的，不过跟这个问题没关，具体在模块的Readme中提到了），于是看了v8内部对async/await的实现，并没有什么发现有跟子进程有什么关系，但是却让我的思路多了一条路，原来我之前用的Promise一直是bluebird的，并没有使用js原生的Promise，于是我通过原生的promise再来执行之前使用bluebird做的异步调用，这次果然也是卡主了，甚至是这样不是异步的操作调用了Promise都会卡主：

	new Promise(function(resolve,reject){
	        resolve(1);
	}).then(function(data){
		console.log(data);
	})

这个时候我意识到，这个问题可能是在Promise身上，于是我查了Promise的规范文档，这上面有这样一句话：

	promise.then(onFulfilled, onRejected)

	2.2.4 onFulfilled or onRejected must not be called until the execution context stack contains only platform code. [3.1].

	Here “platform code” means engine, environment, and promise implementation code. In practice, this requirement ensures that onFulfilled and onRejected execute asynchronously, after the event loop turn in which then is called, and with a fresh stack. This can be implemented with either a “macro-task” mechanism such as setTimeout or setImmediate, or with a “micro-task” mechanism such as MutationObserver or process.nextTick. Since the promise implementation is considered platform code, it may itself contain a task-scheduling queue or “trampoline” in which the handlers are called.

这段规范比较晦涩，不过可以总结出一点，setTimeout和setImmediate属于`macro-task`，而promise的决议回调以及`process.nextTick`的回调则是在`micro-task`中执行的，于是我在v8.h中搜索关于`microtask`的关键词，果然被我找到了一个方法`Isolate::RunMicrotasks`，这个时候我赶紧在我的代码中，也就是子进程`begin_uv_run`函数改成这样:

	bool more;
    do {
        more = uv_run(loop, UV_RUN_ONCE);
        if (more == false) {
            more = uv_loop_alive(loop);
            if (uv_run(loop, UV_RUN_NOWAIT) != 0)
                more = true;
        }
        Isolate::GetCurrent()->RunMicrotasks();
    } while (more == true);
    _exit(0);
    
这个时候，await后面的语句也执行了起来，Promise也不会出现then后的语句不执行的情况了，但却发现`process.nextTick`还是不能执行，于是我到了node的内核中寻求结果，看了一番恍然大悟，原来node的nextTick是自己实现的，并不在`micro-task`中只是，通过代码的方式实现了标准中的执行顺序。下面我们通过node的源码来解释一番这其中的问题以及我通过对这些的了解后做出的最后解决方案。

####node中对macrotask和microtask的调用实现


要了解这个我们首先来看看libuv的启动函数uv_ru那种的代码：
	
	...
	uv__update_time(loop);
	uv__run_timers(loop);//处理timer

	...

	uv__io_poll(loop, timeout);//处理io事件
	uv__run_check(loop); //处理check回调
	...

	if (mode == UV_RUN_ONCE) {  						uv__update_time(loop);
		uv__run_timers(loop);//处理timer
	}

可以从上面看到，主要是三个大事件的顺序，timer，io，check这样的顺序，timer当然就是调用我们`setTimeout`注册回调所用，io自然就是处理我们注册的一些异步io任务，比如fs的读取文件，以及网络请求这些任务。而check中通过src/env.cc中的代码

	uv_check_start(&immediate_check_handle_, CheckImmediate);
	
注册了调用`setImmediate`回调方法的`CheckImmediate`函数。好了现在，`setTimeout`和`setImmediate`都找到了出处，那`process.nextTick`和`Promise.then`呢？这个答案就在`uv__io_poll`中，因为我们所有的io的回调函数最后都是通过 src/node.cc中的函数`InternalMakeCallback`完成的，在其中通过这样的语句来完成整个回调函数的调用过程:

	...
	InternalCallbackScope scope(env, recv, asyncContext);
	...
	ret = callback->Call(env->context(), recv, argc, argv);
	...
	scope.Close();
	
其中的`scope.Close()`是执行`process.nextTick`和`Promise.then`的关键，因为它会执行到代码：
	
	....
	 if (IsInnerMakeCallback()) {
	 	//上一个scope还没释放不会执行
    	return;
	}
	Environment::TickInfo* tick_info = env_->tick_info();

	if (tick_info->length() == 0) {
		//没有tick任务执行microtasks后返回
    	env_->isolate()->RunMicrotasks();
	}
	...
	if (tick_info->length() == 0) {
    	tick_info->set_index(0);
    	return;
	}
	...
	if (env_->tick_callback_function()->Call(process, 0, nullptr).IsEmpty()) {
		//执行tick任务
    	failed_ = true;
	}
	
从上面我们可以知道，在io任务注册的callback执行完了以后便会调用tick任务和microtasks，其中`env_->tick_callback_function()`就是lib/internal/process/next_tick.js中的函数`_tickCallback`，其代码:

	do {
		while (tickInfo[kIndex] < tickInfo[kLength]) {
			...
			_combinedTickCallback(args, callback);//执行注册的回调函数
			...
		}
		...
		_runMicrotasks();//执行microtasks
		...
	}while (tickInfo[kLength] !== 0);
	
可以看到在执行完`process.nextTick`注册的所有回调后，就会执行`_runMicrotasks()`来执行microtask。这里我不禁产生了疑惑，回调我也执行了啊，为何没有执行`process.nextTick`和microtask，唯一不会执行的情况只能发生了在这里：

	if (IsInnerMakeCallback()) {
	 	//上一个scope还没释放不会执行
    	return;
	}
	
带着这个明确的目的，我找到了原因所在，在src/node.cc中通过以下代码来执行js代码的：

	{
    	Environment::AsyncCallbackScope callback_scope(&env);
    	env.async_hooks()->push_async_ids(1, 0);
    	LoadEnvironment(&env);//在这里执行js
    	env.async_hooks()->pop_async_id(1);
	}
		
在AsyncCallbackScope对象的构造函数中会执行如下语句:
	
	env_->makecallback_cntr_++;

而`IsInnerMakeCallback`判断标准就是`env_->makecallback_cntr_>1`，在`callback_scope`析构时会将该值复原，但是我们的子进程在js执行中就分配出来了，并且通过`uv_run`后直接就exit所以并没有机会析构该对象，当然无法调用tick函数和microtask。不过肯定有读者现在产生疑惑了，那假如我不注册io事件 只执行`process.nextTick`和`Promise.then`呢，从上面讲解来看岂不是不能执行，但是我明明执行了的啊，莫急各位看官，因为还有个地方我还没说到，就是node的js启动文件lib/internal/bootstrap_node.js中的命令行交互式启动使用的`evalScript`方法还是直接文件启动的`runMain`中都会在最后执行到`_tickCallback`，也符合js语句执行也是macrotask的一种，在执行完js语句后第一时间执行microtask的原则。所以这个问题的结果就不言而喻了：

	(function test() {
    	setTimeout(function() {console.log(4)}, 0);
    	new Promise(function executor(resolve) {
        	console.log(1);
        	for( var i=0 ; i<10000 ; i++ ) {
            	i == 9999 && resolve();
        	}
        	console.log(2);
    	}).then(function() {
        	console.log(5);
    	});
    	console.log(3);
	})()

首先js先执行所以肯定1，2，3是按顺序执行，而js执行到最后一步就是`_tickCallback`，所以就是5，而执行完了js以后`uv_run`，自然就是执行timer，当然在node中setTimeout的时间为0时，实际为1，所以在第一次调用`uv__run_timers(loop);`不一定会执行，不过不影响这个函数的结果为 1,2,3,5,4。而如果是这样:

	(function test() {
    	setTimeout(function() {console.log(1)}, 0);
    	setImmediate(function() {console.log(2)});
	})()
	
顺序就是不确定的，理由已经讲过了就是第一次timer的调用对time为1的执行与否是不确定的。

清楚了为什么不执行的原因后解决该问题的方法就已经出来了，有两个方法，一个是等js执行完了以后，再分出子进程，可以通过注册了一个timer任务来做，另外一个自然就是在里面分出，但是自己来做 `tick`，我选择了第二个方式，比较简单粗暴，直接通过在子进程的函数中这样写：
	
	bool more;
    do {
        more = uv_run(loop, UV_RUN_ONCE);
        if (more == false) {
            more = uv_loop_alive(loop);
            if (uv_run(loop, UV_RUN_NOWAIT) != 0)
                more = true;
        }
       v8::HandleScope scope(globalIsolate);
    	Local<Object> global =  globalIsolate->GetCurrentContext()->Global();
    	Local<Object> process = Local<Object>::Cast(global->ToObject()->Get(String::NewFromUtf8(globalIsolate, "process")));
    	Local<Function> tickFunc = Local<Function>::Cast(process->ToObject()->Get(String::NewFromUtf8(globalIsolate, "_tickCallback")));
    	tickFunc->Call(process,0,NULL);
    } while (more == true);
    _exit(0);
    
这样就不会再有问题了，通过`_tickCallback`将tick回调和microtask都执行了。


#### 总结

通过这个模块的开发，了解到了`microtask`和`macrotask`的概念并清晰了了解了各个方法的执行顺序，也算是收获满满了。有想法去实行才能获得成长真是真知灼见啊。
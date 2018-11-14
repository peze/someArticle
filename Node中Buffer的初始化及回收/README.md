### Node中Buffer的初始化及回收

node中的buffer相信大家都不会陌生，毕竟这个东西是node的核心之一，我们读写文件，网络请求都会用到它。不过，之前我虽然一直在用这个东西，却没关心过他的实现，只知道通过buffer分配的内存占用的不是v8的heap上的内存，存在于newSpace和oldSpace之外，所以可以用它来进行一些大段内存的操作，但是却从没关心过它是如何分配内存，又是什么时候被回收这些问题。在一次有幸跟我神交已久的一位老哥的交流中，提起了这个问题才意识到自己这一块上确实存在盲区，于是专程去node源码（v8.1.4）中去寻找了一番，也算是颇有所得，所以专门写一篇文章记录和分享一下。

#### buffer的初始化

首先，我们可以从`lib/buffer.js`中，我们可以通过Buffer函数的代码往下追溯，发现Buffer的生成都是通过`new FastBuffer`来生成的，而FastBuffer我们可以看到代码中是这样实现的：

	class FastBuffer extends Uint8Array
	
这是继承自一个`Uint8Array`这个v8内部定义为`TYPE_ARRAY`的类型，从v8在`v8/src/api.cc`的`TYPED_ARRAY_NEW`宏实现中我们可以看到，类似`Uint8Array`的`TYPE_ARRAY`都是通过`ArrayBuffer`来初始化的。

#### ArrayBuffer的实现

那么既然Buffer用的是v8内部的对象`ArrayBuffer`，那为什么buffer分配的内存并不会统计到v8的heap中呢？这个问题需要我们通过观察`ArrayBuffer`是如何实现的，这里我们可以通过`src/node_buffer.cc`中的`Buffer::New`的代码来解释：

	MaybeLocal<Object> New(Environment* env, size_t length) {
		//判断是否能生成
        ...
        data = BufferMalloc(length);

        Local<ArrayBuffer> ab =
        ArrayBuffer::New(env->isolate(),data,length,ArrayBufferCreationMode::kInternalized);
        Local<Uint8Array> ui = Uint8Array::New(ab, 0, length);
        ...
    }
    
从中我们可以看到，node源码中通过`BufferMalloc`分配一段堆内存给初始化`ArrayBuffer`使用，通过分析`ArrayBuffer`的实现过程，我们可以在`v8/src/objects.cc`中的`JSArrayBuffer::Setup`方法中可以看到代码：

	array_buffer->set_backing_store(data);
	
通过这个方法将指向堆内存的指针跟`ArrayBuffer`关联起来，放入`array_buffer`对象的`backingstore`中，所以之前的问题就已经有了答案了，buffer中所使用的内存是通过malloc这样的方式分配的堆内存，只是通过`ArrayBuffer`对象关联的js中使用。

#### Buffer的回收

说起Buffer的回收，我相信已经有聪明的读者想到了，既然是通过js对象`ArrayBuffer`关联到js中使用，那肯定也能通过这个对象利用v8自身的gc来进行回收。没错，对于Buffer的回收也是依赖于`ArrayBuffer`，在其中也是会根据`ArrayBuffer`所在的oldSpace和newSpace的不同进行不同的回收方法，不过都是通过对象`ArrayBufferTracker`来实现的。我们首先来看一下newSpace中的回收方案，在`v8/src/heap/heap.cc`中的`void Heap::Scavenge()`函数，这个是做新生代GC回收的函数，在这个函数中先通过正常的GC回收方案去判断对象是否需要回收，而对于需要回收的`ArrayBuffer`则是通过调用:

	ArrayBufferTracker::FreeDeadInNewSpace(this);
	
来完成的，而这个函数中会轮询newSpace中所有的page，通过每个page中的`LocalArrayBufferTracker`对象去轮询其中保存的每个页中的`ArrayBuffer`的信息，判断是否需要清理该对象的`backingStore`,通过`v8/src/heap/array-buffer-tracker.cc`中函数:

	template <typename Callback>
    void LocalArrayBufferTracker::Process(Callback callback) {
        for (TrackingData::iterator it = array_buffers_.begin();
        it != array_buffers_.end();) {
            old_buffer = reinterpret_cast<JSArrayBuffer*>(*it);
            ...
            if (result == kKeepEntry) {
                ...
            } else if (result == kUpdateEntry) {
                ...
            } else if (result == kRemoveEntry) {
            	 //清理arrayBuffer中backingstore的内存
                freed_memory += length;
                old_buffer->FreeBackingStore();
                it = array_buffers_.erase(it);
            } 
        }
    }
    
而对于oldSpace中，则是通过`v8/src/heap/mark-compact.cc`中的函数`MarkCompactCollector::Sweeper::RawSweep`首先通过代码：
	
	const MarkingState state = MarkingState::Internal(p);

获取page中所有对象标记情况的bitmap，接着通过该bitmap执行函数：

	ArrayBufferTracker::FreeDead(p, state);


通过这个函数来对page上需要释放的`ArrayBuffer`中的`backingStore`进行释放，利用也是page中的`LocalArrayBufferTracker`对象，通过方法：

	template <typename Callback>
    void LocalArrayBufferTracker::Free(Callback should_free) {
        ...
        for (TrackingData::iterator it = array_buffers_.begin();
            it != array_buffers_.end();) {
            JSArrayBuffer* buffer = reinterpret_cast<JSArrayBuffer*>(*it);
            if (should_free(buffer)) {
                freed_memory += length;
                buffer->FreeBackingStore();
                it = array_buffers_.erase(it);
            } else {
                ...
            }
        }
        ...
    }
    
可以看到这部分的代码跟前面几乎是一样的。    

#### 总结

通过对源码的一番窥探，我们可以清楚的了解到了，为什么buffer的内存不存在v8的heap上，而且也知道了，对于buffer中内存的释放，其释放时机的判断跟普通的js对象是一样的。读完有没有感觉对buffer的使用心里有底了许多。
### 前言

本文会通过V8中对String对象的内存分配开始分析，对中间出现的源码进行解读。对heap内存的新生代分配和老生代内存分配的过程解读。首先，我们来看一张流程图，该流程图给出整个分配过程中的前期流程图，其中省略了一些步骤，只给出了关键的步骤。

![image1](http://blog.magicare.me/content/images/2018/09/v8_string_map1-1.jpg)

### 从String::NewFromUtf8开始

我们从String::NewFromUtf8这个函数开始，首先我们来看一下使用，从samples/hello-world.cc中我们可以看到
	
	Local<String> source =
        String::NewFromUtf8(isolate, "'Hello' + ', World!'",
                            NewStringType::kNormal).ToLocalChecked();
                            
这里出现了一个isolate的指针，是通过

	Isolate* isolate = Isolate::New(create_params);
	
语句产生的，这个isolate在v8中是非常重要的一个对象，我感觉类似于句柄的作用，一个v8执行实例几乎所有信息都在上面，包括heap，threadData等等重要信息，这里主要不讲这个就先跳过，接下来的第二个参数，就是一个字符串，第三个参数表示的是String的类型，在源码中分为kNormal和kInternalized两个类型，等会让我们还会讲到他们。

NewFromUtf8方法在src/api.cc中定义如下：
	
	NEW_STRING(isolate, String, NewFromUtf8, char, data, type, length);
  	
NEW_STRING的宏定义就在上面代码的上方,主要流程代码如下：
	
	i::Handle<i::String> handle_result =                                   \
        NewString(i_isolate->factory(), type,                              \
                  i::Vector<const Char>(data, length))                     \
            .ToHandleChecked();                                            \
    result = Utils::ToLocal(handle_result);      
    
在这里我们可以看到函数是通过NewString方法来获取到string的handle，其中isolate->factory()返回的i::Factory对象在代码中的注释 

	Interface for handle based allocation.
	
由此我们可见这个对象包含了所有分配内存并生成对应handle对象的方法。下面是NewString的代码：

	if (type == v8::NewStringType::kInternalized) {
		return factory->InternalizeUtf8String(string);
	}
	return factory->NewStringFromUtf8(string);

这里出现了我们之前提起过的StringType,从注释里，我们可以发现

	kNormal：Create a new string, always allocating new storage memory.
	kInternalized：Acts as a hint that the string should be created in the old generation heap space and be deduplicated if an identical string already exists.
	
kNormal的情况下会创建一个新的string，并且一定会分配内存。而kInternalized则先在老生代（不懂新生代老生代的可以参考这篇文章[浅谈V8引擎中的垃圾回收机制](https://segmentfault.com/a/1190000000440270)）的StringTable(存在Heap中，通过factory对象取得)中搜索是否已经有entry存在，如果存在则直接返回，如果不存在就在老生代中分配内存生成。而KNormal的情况是调用方法NewStringFromUtf8，该方法的原型如下：
	
	MaybeHandle<String> NewStringFromOneByte(
      Vector<const uint8_t> str, PretenureFlag pretenure = NOT_TENURED)
      
该原型中可知，PretnureFlag为 NOT_TENURED,我们可以看看这个flag的的注释：
	
	A flag that indicates whether objects should be pretenured when allocated (allocated directly into the old generation) or not (allocated in the young generation if the object size and type allows).
	
从注释里和flag的名字中我们都可以判断出，这是一个判断分配内存是 长存的（TENURED）还是非长存（NOT_TENURED）的，也就是新生代中还是老生代中。后面的方法Heap::SelectSpace中很明确的说明了这一点：
	
	static AllocationSpace SelectSpace(PretenureFlag pretenure) {
    	return (pretenure == TENURED) ? OLD_SPACE : NEW_SPACE;
    }
    
所以我们可以看出，在kNormal的情况下新建立的string对象都是在新生代的内存区中。这里我们直接从NewStringFromOneByte中的代码流程来说明分配内存的机制，忽略InternalizeUtf8String方法。因为后面其实是殊途同归，不过InternalizeUtf8String中有很大一部分代码是在StringTable中寻找是否已经存在了这个String，所以不太适合我们主题。

### Factory::NewStringFromUtf8初现端倪

我们首先来看一下NewStringFromUtf8的源码：

	// 首先查看字符串是否为一个全是ASCII符号的字符串
	....
	if (non_ascii_start >= length) {
		//如果字符串为ASCII字符串，我们就不需要将UTF8字符串转为ASCII字符串
		return NewStringFromOneByte(Vector<const uint8_t>::cast(string), pretenure);
	}
	//非ASCII字符串则需要转义
	Access<UnicodeCache::Utf8Decoder>
      decoder(isolate()->unicode_cache()->utf8_decoder());
      ...
      ASSIGN_RETURN_ON_EXCEPTION(
      isolate(), result,
      NewRawTwoByteString(non_ascii_start + utf16_length, pretenure),
      String);
      ...


从注释中我们可以看出，在分配内存时会检查是否为ASCII字符串，如果不是，则需要生成一个decoder的对象来处理，在生成了decoder对象后将utf8的字符串转化成ASCII，然后再生成了Handle对象后再转化回UTF8字符串。不过，不管是什么类型在内存分配方面则是一样的，所以我们直接挑选NewStringFromOneByte来继续深入分析。

##### NewStringFromOneByte

在NewStringFromOneByte方法中，代码如下，我只列出了最重要的两个方法：

	//处理length为0和为1的情况
	...
	//为handle分配内存
	Handle<SeqOneByteString> result;
	ASSIGN_RETURN_ON_EXCEPTION(
		isolate(),
		result,
		NewRawOneByteString(string.length(), pretenure),
		String);
	...
	//将字符串复制进分配的内存
	CopyChars(SeqOneByteString::cast(*result)->GetChars(),
            string.start(),
            length);
	return result;
	
从上面的代码和注释中我们可以看出整个流程，首先通过宏定义如下： 

	do {                                                               \
		if (!(call).ToHandle(&dst)) {                                    \
		  DCHECK((isolate)->has_pending_exception());                    \
		  return value;                                                  \
		}                                                                \
	} while (false)
	
可以看出，这个中主要的逻辑在call里面，也就是NewRawOneByteString函数。而下面对生成的result赋值，下面的copyChars函数就是把string中的字符复制进分配出来的内存，很简单也跟我们主题无关就略过不表了。我们主要来看上面的函数NewRawOneByteString，这个函数的源代码如下：

	//检查length是否超过最大阈值，检查length是否大于0
	...
	//分配内存宏方法
	CALL_HEAP_FUNCTION(
		isolate(),
		isolate()->heap()->AllocateRawOneByteString(length, pretenure),
		SeqOneByteString);
    
首先我们来看CALL_HEAP_FUNCTION这个宏，源码太长我们就不贴了，它主要是执行isolate()->heap()->AllocateRawOneByteString(length, pretenure)这个FUNCTION_CALL，在执行完了以后如果未分配成功，则进行垃圾回收，然后再调用FUNCTION_CALL，在未成功的时候会再重试一次，这里为什么会是尝试两次呢，注释中给出了解释：
	
	Two GCs before panicking.  In newspace will almost always succeed.
	
在恐慌之前先做两次GC，这样在新生代中基本都会成功。这个恐慌我觉得用的很有意思，因为在两次失败以后，系统会做调用一个函数CollectAllAvailableGarbage，之前的两次GC调用的是CollectGarbage函数，从这两个函数的名字我们就可以看出，第二个函数做的操作应该是比较大的一次GC，他内部主要是调用CollectGarbage对老生代进行GC，就会涉及Mark和Compact，从之前的文章中我们可以意识到，这样的操作，甚至会造成工作线程的暂停，所以恐慌一词用在这里很传神。
GC方面的代码不是我们这次工作的主要内容，所以这里简要的叙述一下，我们接下来看这次的主题函数FUNCTION_CALL，也就是Heap::AllocateRawOneByteString函数。

#### 从Heap::AllocateRawOneByteString进入分配核心逻辑

首先我们来看流程图：
![image2](http://blog.magicare.me/content/images/2018/09/v8_new_string2.jpg)
从上图我们可以看出，首先通过之前给出的string的length参数算出应该分配的内存空间大小，SizeFor的代码如下：
	
	return OBJECT_POINTER_ALIGN(kHeaderSize + length * kCharSize);
	
这个OBJECT_POINTER_ALIGN的宏定义是将分配大小校准，这个手段在类似Linux内核中经常会用到，比如不足一页大小的内存，扩展到一页，这样可以方便cpu缓存的存取，增加存取速度。接下来的方法SelectSpace我们之前已经聊过了，这里就不再赘述了。这个方法中最核心方法（也是跟我们主题关联最紧的方法）就是Heap::AllocateRaw方法，这个我们等会儿来说。先说最后初始化这个Handle的方法，代码如下:

	//部分初始化需要的object
	result->set_map_after_allocation(one_byte_string_map(), SKIP_WRITE_BARRIER);
	String::cast(result)->set_length(length);
	String::cast(result)->set_hash_field(String::kEmptyHashField);
	
第一个set_map函数是把one_byte_string_map将map加入到对象中，这个Map应该就是String对象的HiddenClass，里面存储了String对象的方法和属性以及其偏移，这样做最大的好处就是String对象基本都一样，对象变动的机会不大，很容易利用到InlineCache的优化（HiddenClass以及InlineCache的文章[Hidden Classes](https://www.understandingv8.com/en/hidden-classes.html)）。后面两个set_length方法和set_hash方法就基本跟他的名字一样，就不必赘言了。
我们现在来重点讨论一下Heap::AllocateRaw,首先从流程图中我们可以看到有五种情况，对应着四个函数，这里我们主要讲新生代和老生代的AllocateRaw方法。

### 分配核心之NewSpace::AllocateRaw

我们首先来讲新生代的内存分配方法，下面是NewSpace::AllocateRaw的流程图：

![image3](http://blog.magicare.me/content/images/2018/09/v8_string_new3.jpg)

这里在32位机器且alignment是kDoubleAligned的情况下会使用函数NewSpace::AllocateRawAlign,这个函数跟NewSpace::AllocateRawUnaligned其实是差不多的，不过会计算一下需要多少大小来填充对象实现Align的过程，所以我们重点分析NewSpace::AllocateRawUnaligned就行了，在NewSpace::AllocateRawUnaligned方法中最重要的函数就是NewSpace::EnsureAllocation方法了，他是确保新生代的内存中有足够的内存来分配给object，至于后面两个方法一个是从当年的顶部生成heapObject，另外一个是将当前的新生代的顶部重置，重置为 当前顶部+这次需要分配的大小，所以我们可以看出整个新生代的内存模型就是不停的在顶部分配内存给对象，是个比较标准的堆内存分配。我们重点来讲解一下
NewSpace::EnsureAllocation的逻辑，下面我们看一下这个方法的主要逻辑：

	Address old_top = allocation_info_.top();
	Address high = to_space_.page_high();
	int filler_size = Heap::GetFillToAlign(old_top, alignment);
	int aligned_size_in_bytes = size_in_bytes + filler_size;

	if (old_top + aligned_size_in_bytes > high) {
	//没有足够的内存page了，增加一个
		if (!AddFreshPage()) {
			return false;
		}
		//让一些allocation observers做统计的方法
		InlineAllocationStep(old_top, allocation_info_.top(), nullptr, 0);
		//重置参数		
		old_top = allocation_info_.top();
		high = to_space_.page_high();
		filler_size = Heap::GetFillToAlign(old_top, alignment);
	}
	
从上面我们可以看出，当前的top+需要分配的内存大于目前空间最大值时，会选择添加一个新的page到新生代的to_space空间中，具体逻辑如下：

	Address top = allocation_info_.top();
	DCHECK(!Page::IsAtObjectStart(top));
	if (!to_space_.AdvancePage()) {
	// 没有更多的page了
		return false;
	}

	// 清除现有页中的残留内存
	Address limit = Page::FromAllocationAreaAddress(top)->area_end();
	int remaining_in_page = static_cast<int>(limit - top);
	heap()->CreateFillerObjectAt(top, remaining_in_page, ClearRecordedSlots::kNo);
	UpdateAllocationInfo();
	return true;
	
上面代码中有两个函数是最核心的，第一个是SemiSpace::AdvancePage,这个函数将Space中的当前页变成下一页来增加空间，如果没有页了或是到达最大页了就会返回false，另外一个函数则是NewSpace::UpdateAllocationInfo函数,这个函数最核心的代码就是这一句：

	allocation_info_.Reset(to_space_.page_low(), to_space_.page_high());
	
他将AllocationInfo对象的top设置为新页的page_low并将high设置为新页的page_high，所以刚刚NewSpace::AddFreshPage的注释中有一个创建一个fillerObject到之前页剩余的空间中的操作。

（其实这里我有一个疑惑，按照这个逻辑那在新生代中生成的对象都是出于to_space中的，但是关于新时代的内存回收文章都写得是新的分配是在from_space中，而在做内存回收时将from_space中不需要回收的放到to_space中去。而且在Heap::Scavenge方法中有这样一段注释和方法

	// Flip the semispaces.  After flipping, to space is empty, from space has live objects.
	new_space_->Flip();
	new_space_->ResetAllocationInfo();

在这里会清空所有to_space里的对象，并将存活的直接放入from_space,然后在后面又将没有升入老生代的object又从新分配进to_space,这个逻辑跟关于V8新生代内存回收的文章中讲的都不太一样，要复杂很多。不过这个跟当下主题也不太符合，只是延伸说一下，下次有时间会再仔细说下这里面的逻辑。）


说回主题来，NewSpace::UpdateAllocationInfo中还有一个方法我们要讲到，因为等会儿的逻辑就是从这里来的，就是设置AllocationInfo的limit的方法NewSpace::UpdateInlineAllocationLimit，这里会根据三种情况设置不同的limit，正常的情况下，也就是在allocation_observers_paused的情况下，limit是等于high的，也就是当前页的末尾；在增量标记的情况下，limit的值为：

	new_limit = new_top + GetNextInlineAllocationStepSize() - 1;
	
最小的limit值是在线性分配被禁用的情况下，就是new_top的值。分析完了NewSpace::AddFreshPage方法后，我们再回到NewSpace::EnsureAllocation方法，添加了新页以后的操作直接在注释中可以看到就不多分析了，我们着重说一下下面的代码：

	if (allocation_info_.limit() < high) {
		Address new_top = old_top + aligned_size_in_bytes;
		Address soon_object = old_top + filler_size;
		InlineAllocationStep(new_top, new_top, soon_object, size_in_bytes);
		UpdateInlineAllocationLimit(aligned_size_in_bytes);
	}
	
这段代码其实有一段注释，大概是说limit<high会在三种情况下发生：

1.线性分配被禁用时

2.在打开增量标记情况下，希望能够分配内存的时候做标记

3.idle scavenge希望能够记录新分配的内存并在大于一个阈值的情况下执行

所以我们可以看出，设置limit的原因最主要是为了为一些allocate observers在allocate发生时执行他们各自对应的AllocationStep方法(比如profile里面的SamplingHeapProfiler用来收集每个函数分配内存大小的监控器就是通过这个调用的)。

到这里新生代的内存分配以及生成object的过程就已经讲完了，接下来我们来说一下老生代的内存分配方法。

### 分配核心之OldSpace::AllocateRaw

因为OldSpace中的AllocateRaw方法并没有重写，是直接使用的父类PagedSpace中的AllocateRaw方法，所以在之前图中，我们直接使用了PagedSpace::AllocateRaw方法来表示。接下来老规矩，我们先看一下老生代内存分配方面的方法流程图：

![image4](http://blog.magicare.me/content/images/2018/09/v8_string_new4.jpg)

从这个图我们就可以看出，对于老生代的分配逻辑上复杂了许多,因为在之前我们在CALL_HEAP_FUNCTION中的注释说了，分配失败以后重试两次GC基本就能让new_space中出现足够的空间分配，但是old_space方面就要复杂的多，让我们来慢慢分析一下其中经历的过程吧。

首先从图中我们可以看出，跟new_space的情况一下，存在着AllocateRawAlign和AllocateRawUnalign的方法，跟之前一样我们直接来分析Paged::AllocateRawUnalign方法，在AllocateRawUnalign代码中先是尝试使用PagedSpace::AllocateLinearly的方法，这个方法跟之前我们在new_space中的方法类似：

	Address current_top = allocation_info_.top();
	Address new_top = current_top + size_in_bytes;
	if (new_top > allocation_info_.limit()) return NULL;

	allocation_info_.set_top(new_top);
	return HeapObject::FromAddress(current_top);
	
可以看一下代码，如果new_top没有大于allocation_info_的limit，那么就能直接从top生成对象即可。如果现有的空间不够，那么就会进行第一次逻辑上稍微简单些的FreeList::Allocate,这里我们涉及到一个在old_space中出现的数据结构，FreeList对象，他是old_space中主要的管理页和分配的函数之一，我们先用一张图来展示它和其他对象的关系，这里面的对象在以后都会遇到：

![image5](http://blog.magicare.me/content/images/2018/09/pageSpace-1.jpg)

这里多说两句来解释一下 Page，FreeListCategory以及FreeSpace的关系类似于Linux内存中三层页表的关系，Page对象其实起到一个全局页目录的作用，而FreeListCategory则是一个二级页目录，而FreeSpace是一个HeapObject的派生类，实际就是我们要存储信息的内存位置，可以理解为一个直接页表项的关系，这里V8的内存结构里比较巧妙，他通过将FreeListCategory按类型分类，然后对不同的类型的FreeListCategory对象中的FreeSpace的内存大小是不一样的，有点类似于控制二级页目录的位数来控制最后页表项大小的方式。

讲完了这些对象的关系，我们就开始说起函数FreeList::Allocate的过程了。

#### FreeList::Allocate

这段代码前，有一段注释能说明为什么oldSpace的分配内存会复杂

	该函数会做一个分配在oldSpace的freeList上，如果成功则直接使用liner allocation将对象返回并设置allocationInfo的top和limit。如果失败，则需要通过GC的方式来回收内存再分配，或者直接拓展一个新的Page给oldSpace;
	
结合我们的流程图大家就可以看到，这是第一次分配未成功的情况，后面两次一次是通过GC的方式来做，第二次则是直接expand一个page。在FreeList::Allocate方法中首先执行两个函数PagedSpace::EmptyAllocationInfo以及Heap::StartIncrementalMarkingIfAllocationLimitIsReached，简单介绍一下就行，PagedSpace::EmptyAllocationInfo方法是将刚刚liner allocation方法中不够用但是又剩余的内存返回到free_list中，并且标记为未使用的空间，这样在GC扫描时会忽略这段内存。至于Heap::StartIncrementalMarkingIfAllocationLimitIsReached函数则是在判断了当前Heap的达到IncrementalMarkingLimit以后判断是立刻开始增量标记的GC任务，还是将该任务加入调度序列，还是不进行。这个函数的判断涉及很多Heap对象里的属性，具体是如何判定我也没有完全搞清楚，但是根据后面的PagedSpace::SlowAllocateRaw方法中使用GC来分配内存的行为，所以这个函数我个人判断是为这次分配失败以后做准备的。接下来的FreeLIst::FindNodeFor是这个函数的核心操作了，我们重点来看一下这个函数，因为没有在图中表示这个函数的内部流程，我们先贴出他的代码来讲解：

	//先通过快速分配来找到适合这个大小的最小FreeListCategory
	//这个操作只需要花费常数时间
	FreeListCategoryType type =
	SelectFastAllocationFreeListCategoryType(size_in_bytes);
	for (int i = type; i < kHuge; i++) {
		node = FindNodeIn(static_cast<FreeListCategoryType>(i), node_size);
		if (node != nullptr) return node;
	}

	// 如果上面的没找到，则通过找Huge类别的FreeListCategory 
	// 该时间是线性增加的，取决于huge element的数量
	node = SearchForNodeInList(kHuge, node_size, size_in_bytes);
	if (node != nullptr) {
		DCHECK(IsVeryLong() || Available() == SumFreeLists());
		return node;
	}

	//如果分配的大小需要huge类型的FreeListCategory来分配，但是却找不到，直接返回null
	if (type == kHuge) return nullptr;

	// 否则找寻最适合该大小的FreeListCategory.
	type = SelectFreeListCategoryType(size_in_bytes);
	node = TryFindNodeIn(type, node_size, size_in_bytes);

	DCHECK(IsVeryLong() || Available() == SumFreeLists());
	return node;
	
之前我们讲到过FreeListCategory，他有不同的类型，我们可以从源码中看到：
	
	enum FreeListCategoryType {
		kTiniest,
		kTiny,
		kSmall,
		kMedium,
		kLarge,
		kHuge,

		kFirstCategory = kTiniest,
		kLastCategory = kHuge,
		kNumberOfCategories = kLastCategory + 1,
		kInvalidCategory
	};
	
不同的类型之间对应了不同的FreeSpace大小，而FreeList::SelectFastAllocationFreeListCategoryType则通过需要分配的size来确定从哪个FreeListCategory开始寻找足够空间的Node，我们接着来看FreeList::FindNodeIn函数，首先来看他的源码：

	FreeListCategoryIterator it(this, type);
	FreeSpace* node = nullptr;
	while (it.HasNext()) {
		FreeListCategory* current = it.Next();
		node = current->PickNodeFromList(node_size);
		if (node != nullptr) {
			Page::FromAddress(node->address())
				->remove_available_in_free_list(*node_size);
			DCHECK(IsVeryLong() || Available() == SumFreeLists());
			return node;
		}
		RemoveCategory(current);
	}
	return node;
	
从代码中我们可以看到，FreeList会对当前type的FreeListCategory生成一个Iterator的迭代器，会迭代在该FreeList的不同page的该类型的FreeListCategory，直到通过FreeListCategory::PickNodeFromList找到一个可用的FreeSpace为止，如果找到了该node则会减去该页上node的size大小的可用空间，而如果整个FreeListCategory都找不到可用空间，则直接从FreeList的该FreeListCategory链表中去掉该项。从FreeListCategory::PickNodeFromList的代码中:

	DCHECK(page()->CanAllocate());
	FreeSpace* node = top();
	if (node == nullptr) return nullptr;
	set_top(node->next());
	*node_size = node->Size();
	available_ -= *node_size;
	return node;

我们可以看出，在FreeListCategory中，FreeSpace也是一个链表，而top则是最近一次分配出来的FreeSpace，所以在分配成功后，除了在该FreeListCategory减去可用的内存大小以外，还要重新set_top。

当type为Huge或者之前type<Huge中都没有找到可用的node，则会使用FreeList::SearchForNodeInList在kHuge中去查询kHuge中可用的FreeSpace，从函数的代码中我们可以看到：

	FreeListCategoryIterator it(this, type);
	FreeSpace* node = nullptr;
	while (it.HasNext()) {
		FreeListCategory* current = it.Next();
		node = current->SearchForNodeInList(minimum_size, node_size);
		if (node != nullptr) {
			Page::FromAddress(node->address())
				->remove_available_in_free_list(*node_size);
			DCHECK(IsVeryLong() || Available() == SumFreeLists());
			return node;
		}
		if (current->is_empty()) {
			RemoveCategory(current);
		}
	}
	return node;

这个函数的大概逻辑跟之前其实是比较像的，只是多了一个minimum_size，这个大小是真正需要的大小，而node_size则是huge类型的FreeListCategory的node大小，之所以需要保留这个大小，主要是避免大量空间的浪费，后面我们会说到。再来看这个函数中的核心函数FreeListCategory::SearchForNodeInList：

	FreeSpace* prev_non_evac_node = nullptr;
	for (FreeSpace* cur_node = top(); cur_node != nullptr;
		cur_node = cur_node->next()) {
		size_t size = cur_node->size();
		if (size >= minimum_size) {
			DCHECK_GE(available_, size);
			available_ -= size;
			if (cur_node == top()) {
				set_top(cur_node->next());
			}
			if (prev_non_evac_node != nullptr) {
				prev_non_evac_node->set_next(cur_node->next());
			}
			*node_size = size;
			return cur_node;
		}

		prev_non_evac_node = cur_node;
	}
	return nullptr;
	
通过代码我们可以看到，在通常情况下，对象需要size一般是小于huge类型的FreeListCategory的对象size的所以逻辑和之前的一样。但是如果超出了大小，则会继续往下寻找，一直找到合适大小的node然后将这个node从链表中排除，而不改变top。所以我们回想之前在FreeList::FindNodeFor中的注释，就可以知道为什么说之前那些type的查找为O(1),而这个type为O(n)了。如果这样并不能查出合适的node，那么就会通过FreeList::SelectFreeListCategoryType方法定位更准确的type，总的来说就是定位一些需要内存更小的对象，至于FreeList::TryFindNodeIn方法跟之前的方法类似，最后一步其实使用的也是FreeListCategory::PickNodeFromList函数，O(1)查找方法，所以也不需赘言。

在分配完成后，如果分配成功则会通过PagedSpace::AccountAllocatedBytes方法将老生代中分配状态信息中的已用size加上刚分配的对象大小，但是在禁用线性分配的情况下，会通过PagedSpace::Free释放掉没有使用的大小,并且将老生代allocationInfo的top和limit都设置为top加上需要分配对象的大小。而在剩余大小大于一个阈值IncrementalMarking::kAllocatedThreshold且增量标记未完成的情况下，也会通过PagedSpace::Free释放掉没有使用的大小，不过这里会多留出一个liner_size的大小，并且设置allocationInfo的top为top加上需要分配对象的大小，而limit为当前的top再加上liner_size。除了以上两种情况，allocationInfo的top为top加上需要分配对象的大小，而limit为top加上分配的整个对象的大小。

而如果未分配成功的话，从流程图我们可以知晓，接下来会调用函数PagedSpace::SlowAllocateRaw,不过PagedSpace::SlowAllocateRaw函数主要是一个过渡函数，将当前的heap的VMState设置为GC状态，并开始对马上要执行的函数设置一个timer来计时，而马上要执行的函数PagedSpace::RawSlowAllocateRaw才是我们真正需要了解的函数。

#### PagedSpace::RawSlowAllocateRaw

从流程图我们可以看出函数的第一步，如果判断当前的heap在做sweep操作且不是在CompactionSpace中且sweeper本身已经没有task在运行了，则通过MarkCompactCollector::EnsureSweepingCompleted函数等待sweep操作结束再进行下面的操作。在结束了sweep以后，会重新装填FreeList的page，因为此时sweeper线程已经释放了一些object了，这个操作由PagedSpace::RefillFreeList来完成，其代码如下:

	if (identity() != OLD_SPACE && identity() != CODE_SPACE &&
		identity() != MAP_SPACE) {
		return;
	}
	MarkCompactCollector* collector = heap()->mark_compact_collector();
	intptr_t added = 0;
	{
		Page* p = nullptr;
		while ((p = collector->sweeper().GetSweptPageSafe(this)) != nullptr) {
			if (is_local() && (p->owner() != this)) {
				base::LockGuard<base::Mutex> guard(
					reinterpret_cast<PagedSpace*>(p->owner())->mutex());
				p->Unlink();
				p->set_owner(this);
				p->InsertAfter(anchor_.prev_page());
			}
			added += RelinkFreeListCategories(p);
			added += p->wasted_memory();
			if (is_local() && (added > kCompactionMemoryWanted)) break;
		}
	}
	accounting_stats_.IncreaseCapacity(added);

第一行的判断比较简单就不用多说了，我们主要看while循环中的逻辑，Sweeper::GetSweptPageSafe可以得到已经sweep过的page，然后后面的逻辑中先判断是否为CompactionSpace且该page不属于该space，因为我们现在是在OldSpace中，所以这个逻辑我们直接忽略，我们来看下面的逻辑，首先是PagedSpace::RelinkFreeListCategoriesf方法，其执行的代码如下：

	DCHECK_EQ(this, page->owner());
	intptr_t added = 0;
	page->ForAllFreeListCategories([&added](FreeListCategory* category) {
		added += category->available();
		category->Relink();
	});
	DCHECK_EQ(page->AvailableInFreeList(), page->available_in_free_list());
	return added;
	
其中通过Page::ForAllFreeListCategories方法，将遍历Page中的每个可用的FreeListCategory并执行后面的匿名函数(C++的lambda函数)：

	[&added](FreeListCategory* category) {
		added += category->available();
		category->Relink();
	}
	
其中先通过added来加上page中所有FreeListCategory上能用的大小，又通过FreeListCategory::Relink将当前Page所属的PagedSpace的FreeList设置为该FreeListCategory的owner，完成了对FreeListCategory的操作后，再加上page中浪费了的内存量。扫描完所有的sweptPage以后,将最后的add值增加到oldSpace的容量中。在做完这一系列操作以后，会再一次重试通过我们之前讲过的FreeList::Allocate方法再一次分配，如果依然没有成功，这个时候就会调用Sweeper::ParallelSweepSpace，这个函数作用在名字中已经表明了，这是一个做并行sweep PagedSpace的函数,代码如下：
	
	int max_freed = 0;
	int pages_freed = 0;
	Page* page = nullptr;
	while ((page = GetSweepingPageSafe(identity)) != nullptr) {
		int freed = ParallelSweepPage(page, identity);
		pages_freed += 1;
		DCHECK_GE(freed, 0);
		max_freed = Max(max_freed, freed);
		if ((required_freed_bytes) > 0 && (max_freed >= required_freed_bytes))
			return max_freed;
		if ((max_pages > 0) && (pages_freed >= max_pages)) return max_freed;
	}
	return max_freed;
	
很明显，代码会去拉取space中将要被sweep的页，然后调用Sweeper::ParallelSweepPage方法直接做page的sweep，得到的单页释放内存大于需要的内存时或释放页数到达规定页数时则会返回。从流程图中我们可以看出在Sweeper::ParallelSweepPage最重要的就是Sweeper::RawSweep函数，在sweep未完成的时候会调用。Sweeper::RawSweep自然是sweepPage的核心，不过这个函数比较复杂，我们只能根据流程图并提取关键代码大概讲一下逻辑。首先，该函数会通过代码:

	const MarkingState state = MarkingState::Internal(p);
	
获取页中所有对象标记情况的bitmap，接着通过该bitmap执行函数：
	
	ArrayBufferTracker::FreeDead(p, state);
	
这个函数的作用是根据所给page的bitmap将page中已经dead的JSArrayBuffers所有backing store（这里为什么free掉的只有JSArrayBuffer？ArrayBufferTracker中的array_buffers_属性的注释中说这个集合中包含了这个页上被GC移除的raw heap pointers，为什么不直接使用heapObject?猜测是因为JSArrayBuffer每个索引对应HeapObject的一个字节，在Free的时候通过它的特性比较方便，所以GC把需要移除的HeapObject转化为JSArrayBuffer）。
在成功的释放了page中一定的空间以后，会再通过page的bitmap找出所有存活的object，通过这些存活的object找到空闲的内存区间，并通过PagedSpace::UnaccountedFree方法将该空闲内存返回到对应他内存大小的FreeListCategory中去，而在存在有对象在新生代但是引用他的对象在老生代的的情况下，则需要通过RememberedSet<OLD_TO_NEW>::RemoveRange方法来清理掉这个引用的记录（就是之前关于V8 GC文章中提到的写屏障insert的记录）。

在做完了Sweeper::ParallelSweepPage操作后，又得到了新的swept page，所以再一次执行函数PagedSpace::RefillFreeList，执行完成后又一次执行FreeList: Allocate尝试分配。而如果这次失败，程序就会判断可能是页真的不够了，在通过Heap::ShouldExpandOldGenerationOnSlowAllocation判断能扩展页以后，会调用PagedSpace::Expand来扩展页，从流程图中可知PagedSpace::Expand的步骤，先是通过MemoryAllocator::AllocatePage分配出页，接着调用PagedSpace::AccountCommit更新space中的committed_属性，最后将生成的页插入到space中，再增加页后继续尝试FreeList: Allocate，如果这次依然失败（可能是已经到达老生代内存的瓶颈了）则会调用PagedSpace::SweepAndRetryAllocation方法，这个方法相对简单，代码如下，就不讲解了：

	if (collector->sweeping_in_progress()) {
		collector->EnsureSweepingCompleted();
		return free_list_.Allocate(size_in_bytes);
	}
基本都是之前讲过的，这里只是做最后一次尝试。

如果PagedSpace::SlowAllocateRaw分配成功，则需要对分配成功的区域进行一个标记，标记该内存段是刚分配的不需要清理，使用Page::CreateBlackArea来完成。在以上操作成功的返回Object后，就会调用函数PagedSpace::AllocationStep，这个函数我们也不陌生，在新生代中使用过，他会通知space中的各个allocation observers调用各自的AllocationStep方法，做一些统计方面的工作。


### 总结

以上就是整个V8内存分配中的过程，该过程非常复杂，还涉及了很多GC相关的东西，但是V8代码确实是一个精湛的工艺品，里面的函数名都取的让人知道是做什么的，里面的注释也恰到好处，很多时候我陷入困惑的时候总能从一些注释中得到线索，慢慢又顺藤摸瓜的搞出答案，当然这个只是系统的一部分，而且整个系统异常的精密，很多东西不联系其他模块上也无法知道，所以上面也存了一些疑惑的地方。下次有时间，我会再来一探GC的究竟，当然这块更是一个硬骨头，希望能够搞懂~
	

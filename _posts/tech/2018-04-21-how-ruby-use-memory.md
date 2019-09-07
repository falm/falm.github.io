---
layout: post
title: Ruby内存分配
category: 技术
tags: [ruby]
keywords: ruby,memory
---

性能优化是使用Ruby开发web应用中一个比较头痛的问题，因为Ruby本身的执行性能并不高，而且重量级应用Rails使用ruby的方式又是极其消耗内存的。我们知道内存消耗过大对性能有影响，那么内存的消耗是如何影响性能，这就需要对Ruby使用内存的结构和GC的工作方式有比较深入的了解。

### 内存结构

![ruby-memory-structure](http://upload-images.jianshu.io/upload_images/1767848-378fcbec54adfbb4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**1.Ruby管理的 栈和堆**

在Ruby管理的 栈空间中存放的是  当前运行栈的RVALUE指针和值，而堆空间存放的是，RVALUE值，其中值的大小为固定的40bytes ，其结果是 heap 默认包含24个heap page  heap page 默认包括 408个 slot  一个slot 存放一个对象，集RVALUE(Robject RArray..)  （按照Ruby默认初始化10_000个slots 计算）。其中Ruby 2.1 中 默认会多初始化一个heap page 也就是25个heap page，并且堆空间的heap page 是按照  eden 和tomb 两个分区存放的。在运行GC的时候，Ruby会从eden中清空不在使用的heap page，然后移动到 tomb中标记为free。

GC: x: 需要扫描回收的对象数量 
y = x/HEAP_OBJ_LIMIT (GC需要扫描的heap page数)
z = 0.8 * (80% of total heap slot count |GC_HEAP_INIT_SLOTS)

**2.系统管理的堆**

系统管理的堆空间存放的是 数据，即RVALUE中无法存放的的数据，比如一个1K的字符串，RVALUE只能存放引用和元信息和23个字符，剩下真正的数据是存放在系统管理的堆空间的，数据用完后会被GC释放。

### GC触发条件

1.仅剩余20%的free slots时执行GC   (RUBY_GC_HEAP_FREE_SLOTS) 4096
2.mallocate 分配16MB以上内存时执行GC ( RUBY_GC_MALLOC_LIMIT ~ RUBY_GC_MALLOC_LIMIT_MAX 32MBMB区间个字节 具体数字是Ruby根据内存使用情况动态确定的)

### 问题和优化

Ruby的内存使用方式造就了它在多次GC和malloc之后 堆内存中会出现很多的碎片，这些部分的内存是部分被利用上的，然后就会导致堆内存不足需要继续分配内存，然后再产生更多碎片一尺类推。

![malloc_memory_fragmentation](https://upload-images.jianshu.io/upload_images/1767848-9b07bfaf2b474c3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 

Ruby 2.1之后内存heap分为  eden 和 tomb 两个部分，分别存放新生对象和死去的对象，当你创建对象的时候，ruby会从eden space 分配内存，如果eden中的内存不足的话，再从tomb空间中分配内存使用。这么做的原因有两个，其一是通过重复使用 heap space 来减少内存空间的碎片。其二是方便解释器释放整块内存空间。

那么在真正的使用场景下，Ruby仅仅释放了tomb空间10%的内存，因为ruby解释器程序具有马太效应，使用过多对象的应用，在未来还会使用更多的对象，也就是更多的heap space所以，它通过保留80%多的tomb内存空间。这样未来要使用更多的内存空间时，不需要再次分配内存（分配内存消耗资源）。

优化内存的使用主要就是两点，一是添加内存分配的参数，二是调整GC触发的参数。下面的几个Ruby参数就是和内存使用优化相关的参数：

**RUBY_GC_HEAP_INIT_SLOTS**： Ruby初始化slots数量，数量越大后续malloc次数越小

**RUBY_GC_HEAP_GROWTH_FACTOR**：内存分配增长因子

**RUBY_GC_MALLOC_LIMIT**： GC触发条件之一, 分配内存超过限制后运行GC

**RUBY_GC_HEAP_FREE_SLOTS**：对堆空间free slot少于该参数是 触发GC

最后这些参数修改的要点是，需要根据你现有应用的内存使用情况来设置，比如像Rails这样的应用可以在初始化的时候就需要分配很多的内存，那么就可以一开始吧初始化的slots数量调高一些，这样在应用启动时能够减少malloc的数量。
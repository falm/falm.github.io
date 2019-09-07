---
layout: post
title: Rails应用内存优化
category: 技术
tags: [rails,ruby]
keywords: rails,ruby,reduce-memory-usage
---

我们知道Rails应用的内存占用通常都是比较高的，尤其是比较重型的全栈应用内存使用更接近1G（当然同时也包括想sidekiq这样加载整个Rails应用的ruby进程），所以我们通常对应这种情况都采取一种比较tricky的方式，使用像**puma_worker_killer**这样的监控程序，监控rails进程达到一定内存占用后将其重启。也就是说应用一开始的内存占用通常都在100~300MB之间，随着时间的推移进程会创建大量的『对象』内部又会进行数次内存分配和回收，所以内存就会不断飙升。

> 想详细了解Ruby如何分配和使用内存，可以看我的另一篇博客 [ruby内存分配](https://www.jianshu.com/p/e4f184e92375)

在我们知道了基本情况之后，那就该说说正题如何优化Rails的内存占用，解决方案有若种，我们这里讲解一个最容易实施而且见效最快的方式，就是从内存分配入手。ruby使用glibc的malloc(3)进行内存分配，这是一个比较古老的内存分配器，性能比较低分配时会产生大量碎片。所以真的这一点，现在由很多性能比较出色由兼容原有API和以及被验证过的特性的新分配器，如jemalloc和tmalloc，这里我们就使用jemalloc作为Ruby应用的内存分配器，来看看能达到什么样的优化效果。

### jemalloc

jemalloc是facebook出品的，最早用于FreeBSD中的内存分配器，后来像firefox从3.0也开始使用它，redis从2.4之后默认在linux上使用jemalloc。既然有这么多性能敏感型的软件都使用了jemalloc那它一定有过人之处。

jemalloc的特别之处在于它融合了其他内存分配技术的优势，并且采用多级内存分配，线程池缓存tcache还有划分内存区来减少线程间锁的争用。

jemalloc结构：

![结构图](http://upload-images.jianshu.io/upload_images/1767848-7c19840edf678b25.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**多级内存分配** ：jemalloc根据对象的大小，将其归为划分为 small object, large object和huge object三类

- Small objects的大小以8字节，16字节，32字节等分隔开的，总体小于页。
- Large objects的大小以分页为单位，等差间隔排列，总体小于chunk。
- Huge objects的大小是chunk大小的整数倍。

**Arena**: jemalloc 没有像malloc那样对内存的划分都几种在一个区域中管理，而是使用多个小块的内存区域来分别管理，内个小块称为"arena" 。

**线程池缓存thread cache** ：tcache是分配线程的缓存空间，jemalloc使用tcache来减少线程内存分配中锁竞争，从而提升分配效率，每个tcache对应一个arena。

![image](http://upload-images.jianshu.io/upload_images/1767848-0261f8df12075147.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 优化效果

这一步我们来看看应用Jemalloc到我们的ruby进程中到底能有多大的提升呢。

**安装** 

```shell
> sudo apt-get install libjemalloc-dev
> rvm reinstall 2.4.1 -C --with-jemalloc
```

我们选择在2.4.1版本上进行测试。除了上面这种安装方式外，也可以通过gem包来安装[**jemalloc-rb**](https://github.com/kzk/jemalloc-rb)

**内存使用**

我们在同一个应用运行的两台服务器中的一个上面安装的jemalloc，而另一套作为对照组没有安装。

这里就贴出性能差距最明显的一个进程  rails应用的sidekiq进程

没有使用jemalloc的服务器上面的sidekiq进程

![sidekiq without jemalloc](https://upload-images.jianshu.io/upload_images/1767848-b60754de527e6a5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用jemalloc的服务器上面的sidekiq进程

![sidekiq with jemalloc](https://upload-images.jianshu.io/upload_images/1767848-414f35b0fa46af19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到差距是很明显的，当然不是每个进程都有这样的优化效果，这个我们总结的时候再说。

![image.png](https://upload-images.jianshu.io/upload_images/1767848-61eb1dbb8c02c94c.png?imageMogr2/auto-orient/strip%7CimageView2/1/w/640/h/480)

### 总结

从上面的jemalloc的前后对比图中我们能够看出jemalloc的优化还是有明显效果的。至于为什么sidekiq和puma之前的差距这么大，这就引出了，其实jemalloc仅是从内存分配的程度去优化和改进内存使用和性能，在你的应用程序大量产生对象，并且长久运行下去的情况时，效果就比较明显，而如果应用本身做的就比较精简，从程序角度优化的比较好的话，jemalloc的提示就不明显。所以说jemalloc可以为你的应用内存性能解燃眉之急，但是从系统性的角度出发，还是从自身出发优化好应用本身的性能。
---
layout: post
title: 分布式锁
category: 技术
tags: [分布式系统,分布式锁]
keywords: 分布式系统,互斥
---

今天写点分布式锁相关的东西

## 单机锁与分布式锁
单机环境，当共享资源自身无法提供互斥的能力时，就需要引入第三方的互斥能力，来避免多线程/进程对共享资源的读写所造成的不一致。
那么单机锁一般是由内核或者有互斥功能的类库提供

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghl3khi08dj311i0cmwfi.jpg)
由此我们抽象一下分布式锁的概念，首先分布式锁需要一个资源，这个资源能够提供并发控制，并输出一个排他性的状态，也就是 lock = resource + concurrency control +ownership(展示所有) 
已单机为了：
spinlock = bool + cas (optimistic lock)
mutex  = bool + cas + notify  (pessimistic lock)
spinlock 和mutex都是一个bool资源，通过原子的CAS指令，设置0为1，成功的话持有锁，失败就不持有锁，如果不提供所有权展示，AtomicInteger也是一个通过资源 (integer) + CAS但是不明确提示所有权，隐藏不会被视为一种锁，当然，我们可以将『展示所有权』更多的视为某种服务提供的形式的包装
分布式锁的特性：
| 资源 | 互斥访问 | 可用性  |
| ---- | -------- | ------- |
| 文件 | 创建文件 | TTL     |
| kv   | put kv   | session |

## 分布式锁的分类
根据锁定资源的安全性本身，我们将distributed lock 分为两个大类
1. 基于异步复制的方案 ,例如：mysql,tair,redis etc
2. 基于paxos等共识的分布式一致性系统，例如 zk，etcd,consul
基于异步复制的分布式系统，存在数据丢失的（丢失锁）的风险，这样对于被锁定的资源来说不够安全，其往往是通过ttl的机制承担细粒度的锁服务，这类系统接入比较简单，适合对时间很敏感，期望设置一个较短的有效期，执行短期任务，锁丢失对业务影响可控

基于paxos等一致性共识协议的分布式系统，通过一致性协议保证多副本数据一致性，数据安全性高，系统往往通过lease(租约)等类似机制承担细粒度的锁服务，这类系统有一定的使用门槛，适合对安全性要求较高，期望长期持有锁，不希望发生锁丢失的服务。

## 分布式锁选择 
我们使用分布式锁，来做资源使用的互斥，其主要的目的可大体分为两类
1.效率为目标，2.一致性为目标的

其中效率为目标的部分，我们可以容忍系统在分布式锁服务在短暂的期间内不能对资源进行互斥保护，这个无法是增加一些系统资源的消耗，在SLA可以容忍的范围内
在这个目标中，我们就不需要强调，分布式锁服务的高可用特性，一般利用 数据库（比如 mysql）和redis具备基本的主备failover能力，在master 宕机的时候 数据没有来得及复制到slave，slave就切为master，可以容忍短暂的锁丢失。


另外一个以数据一致性为主要目的使用场景，通常是要有系统状态变更，互斥资源被多个使用方访问，导致系统永久性状态不一致，问题比较严重，所以选取分布式锁方案就要，使用具有分布式一致性协同能力的方案。


## 分布式锁需要注意的问题
锁冲突失效
如下图试想如果两个节点a,b先后尝试想锁服务请求锁占用a节点获得锁后，突然因为GC停顿，或者网络延迟，导致处理时间超过了，持有锁的过期时间，b节点在着之后有获得了锁，尝试写入资源，当a恢复处理时任务之间还持有锁，两个不应该操作的执行顺序就发生了颠倒
![lock held by client](https://tva1.sinaimg.cn/large/007S8ZIlly1ghlqtxw2ssj30uk0b4wfj.jpg)
解决这个问题，可能首先会想到让锁的过期时间设置的足够长，超过网络延迟和GC停顿，但是我们不能用拍脑袋的方式去判断网络延迟或者GC最大会有多长时间，而且如果设置过长的超时时间，会明显降低锁服务的吞度量。
最后的解决办法就是映入Fecing机制，在节点每次获得锁的时候分别一个全局唯一有序的版本号，在时检查该版本号是否大于锁上次写入的版本（类似乐观锁）这个解决方案在Google Chubby中有实现来解决这个问题，其他的系统入zookeeper 利用分布式全序关系广播来实现这一点。


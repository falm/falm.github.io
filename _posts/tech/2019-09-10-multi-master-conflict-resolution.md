---
layout: post
title: 多主复制冲突
category: 技术
tags: [分布式系统,数据库]
keywords: 分布式系统,数据库
---

首先要搞清楚为啥要使用多主模式，这样才multi-master数据冲突解决，讨论才有意义。
在有状态的数据集群中，通常采用的模式有：signle-master, master-slave, multi-master, multi-master-slave，几种模式
其中多master集群，是将数据从单一master接受全部请求的模式转换为 两个master节点共同接受请求
我们知道无状态节点很容易通过增加集群机器来进行横向扩展，提高吞吐量，降低单个节点负载。
但是数据节点是有状态的想要通过简单的增加节点，来提高性能就没那么容易了，一般是采用 master-slave做读写分离，读请求由slave处理，这样扩展slave节点的数量就能很快的提示整体的性能。
那么multi master 模式解决的是写请求比较多的场景，单个master节点负载太高或者是多数据中心（异地多活，同城双活）提供系统可用性，总的来数就是，提高可用性和性能两点



## 多Master副本模式下做主要的问题- 数据冲突
试想一下，如果你有笔转账请求，发送到m1节点，然后突然m1节点掉电，你重试请求到m2节点，两次操作的数据就有冲突。
在比如，同一时刻有两个客户端，在两个数据中心对同一个数据进行修改，两个数据中心master进行副本日志同步的时候就会出现冲突。

这种多数据中心冲突的问题，有一种规避的方式是，通过将对同一个数据的操作路由指定到，固定的机房绩效，也就是对请求做路由shard，但是这个仅仅使用某些特定的业务场景，比如twitter和wechat这种就可能不适用，而O2O业务的公司到是能够用得上。
如果不能才用shard逻辑避免冲突，或者是多活shard高保进行切换了切，一样会有冲突的问题要我们解决

通常冲突解决的方式有两大类：一类是N选一，另一类是合并

## LWW(last write wins)
N选一的方案，就是，从一堆冲突的数据中挑选出最后一个提交的数据，然后以这个数据为准，进行持久化
这里就有两个问题，1怎么选出最后的提交的数据，2冲突的其他数据怎么办。

### 精准时间
要确定在副本中复制的数据，的决定时间顺序是非常难的，就是是都开了NTP，精度也是秒级的，还是有可能冲突，另外网络不可靠，节点上的NTP对的时间也不一定及时和准确。但也不是完全没招，Google的 spanner论文中就提过TrueTime API，应中心话的时间服务，提供准确时间（GPS时钟），这东西有没有那么精准先不说，光增加一个时间分配集群成本就够高的了

### 逻辑时间
我们换一种思路想，去顶不了物理上的精准时间，我们就判断，数据发送的先后的逻辑顺序，就行也不关心到物理数据，逻辑时间有名的就是1978年被提出"lamport clock"，它可以表示分布式系统中时间的全序关系，但是不能表示因故关系，也就不能判断两个时间是否是同时发生的(逻辑上同时)，所以要用作冲突检查，需要使用改进的 "vector click"。
向量时钟，简单的来说，就是集群中每个节点有自己维护的数据更新时间，并且与集群中其他节点互为广播，这样，就可以得到，全部节点的向量时钟，以此来判断，当前的数据是否与其他节点的数据有冲突关系（不是因果就是同时发生的）。
在实际应用中，多事会采用时钟向量，变种的版本向量(vector version)（比如亚马逊的Dynamo)。同时
无论是时间向量，还是版本向量都只是冲突检查的手段，具体的冲突解决，还需要再具体的实现中，采用适合的方式，比如下面会说的合并冲突，或是丢弃部分数据。


### 没有银弹
没有银弹，精准的物理时间和因果关系的逻辑时间，都有成本。所以说技术含金量最高的那个不一定最流行，现行采用最多的其实就是，不care时间到底准不准，每天记录上标识一个timestamp或者uuid（只要可以判别顺序），就取最后一个，如果两个相同就随便选一个，像Cassandra，MongoDB都是采用这个短平快的方式。
还有第二个问题，冲突里面的其他数据就当做丢失了，这也是LWW模式的缺点



## 无冲突模型 CRDTs(conflict-free replicated data types)
CRDTs 中文直译的话非常绕口，我就称呼其为无冲突合并解决方案，这个方案的思路，就是如果在特定数据结构下，比如list，set等列表数据结构，冲突其实可以做一个并集运算，或者叫同或运算，这样两边的修改就可以都保留在最终的数据中。采用这种方案的弊端就是，数据复制的工具需要，知道实现，不同数据结构的合并方法，这个就很费事了，而且也不是啥数据结构都能搞的，比较MySQL普通的行记录，就搞不了，值能是特定的nosql数据库，比如redis里面的列表结构或者是riak数据库。




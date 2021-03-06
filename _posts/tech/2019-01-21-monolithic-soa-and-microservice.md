---
layout: post
title: 从IT发展史看单体,SOA与微服务
category: 技术
tags: [闲聊,微服务]
keywords: SOA,microservice
---

今天聊聊老生常谈的应用架构模式，单体，SOA，与微服务。如果仅仅是介绍这三个东西，我觉得也没啥好写的，网上资料一大堆，我自己也不用记录啥。但如果已中外发IT发展史的角度看这三个东西，资料就比较少了，可以记录的东西就有了。

## 20世纪末

信息商业化应用的发展离不开以太网（局域网）和广域网的发展，在80s末期，美国各追求上进的商业公司们都开始购买软件产品（类似财务软件，进销存等等）来帮助更好的运营公司和减少成本开支，有些公司更是构建了自己的IT团队，当然这个主要还是，帮公司里的电脑调试系统，按照软件，还有维护一下公司的内部往来，其中最主要的工作，就是作为甲方与软件提供商进行接洽（IT类小说『凤凰项目』会对此也有过描述）。
为企业提供IT信息话软件的公司，提供的软件在那个时代还都是C/S架构，服务端的架构也是单体的，这种模式一度运转的很好，但是IT行业就是不断向前发展的，尤其是在那个IT技术爆炸的年代，广域网的应用WWW的出现，带领我们进入互联网时代，90s末期，在这个时代背景下，高估值的互联网企业不断上市，让企业家和IT从业者看到信息化的在资本上的吸引力，或主动或被动的，都对一起，花了不少价钱买来的，各种财务和库存管理软件心生不满。
再次背景下 Gartner 96提出了SOA架构的概念，迎来帮助这些企业的内部从不同软件公司买来的，使用不同技术体系实现的软件系统，能够直接互通，成为一个整体为企业服务（可能企业会因为这项改进，就可以向华尔街和投资人说，已经进行了信息化改造，在互联网大热的时候提振股价），当SOA概念开始传播，大型的IT其他开始看上了这块肥肉，分分加入其中（类似如今的工业/产业互联网风口）概念开始落地，SOAP,WSDL,ESB等具体的技术方案开始出现并应用。

稍微总结一下：
SOA的诞生，一是依托了现存大量的企业内部系统，二是WWW推动的信息化浪潮，让企业看到内部的异构系统可以更好的做信息交换来提高信息的生产利用率和效率。
与此同时的我国企业信息化的应用基本可以说是才刚刚起步，没有什么企业已经建立了 异构的IT系统需要互通整合，关心企业信息化的企业甚少。
所以千禧年左右国内SOA基本没有什么应用（这里可能要排除外企）。

## 互联网泡沫后
随着千禧年互联网泡沫的破灭，互联网创业已经不是随便搞个热点网站就能上市割韭菜的时代了，企业中尤其是存活下来比较好的企业，开始丰富自己的产品线护城河，业务上的增加，同时也是对技术提出了更高的要求，单体被拆分成SOA在，互联网企业里面开始进入实践，其中代表性的就是，亚马逊的SOA实践，践行轻量级的SOA技术体系。
这时候SOA从最开始提出为解决企业信息化的问题（内部不同提供商的异构系统通信），转变为互联网企业，解耦技术组织架构的手段，同期的国内基本和我们国工业化一样，是跑步进入信息化，SOA在TOP互联网企业也开始落地
总结下就是：互联网泡沫后到10年前，TOP的互联网企业（无论中美）都开始进入发展期，人员和组织结构都开始不断膨胀，这使得SOA在他们内部开始得到应用来解决技术架构上的问题，SOA也从非IT企业内部异构系统通信的出发点转变为，大小互联网企业内部系统组织划分和交互的问题，可以说这个节点国内是跟上了这次技术演进的步伐。

## 移动互联网兴起
智能手机和应用开启了，近十年爆发式增长的移动互联网，新的领域大家几乎是同一个起跑线，一个App只要解决了人民群众的需求，就能获得付费或者收获很多用户（这也意味着会获得资本的青睐），让移动互联网创业成为热潮，在竞争激烈，热点舒心万变，为了不断的挖掘用户需求，匹配市场走向，创业团队无不采用快速迭代的方式开发产品，但是随着产品用户量不断增加，系统逐渐浮躁，单体应用搞不定，创业团队也开始有模有样的学习TOP企业的做法，实践SOA的改进版本，并与TDD和devops结合，只带15年的时候martin fowler总结了这种模式『微服务』，然后就是大家熟悉的了，『微服务』在那两年火了（无论中外）。
总结下
微服务本身是从中小型移动互联网公司对SOA实践和改进更适合这个时代的快速迭代产品和组织的节奏提来总结而来的，它不是什么突破创新的新的架构模式，而是对过期最佳实践的提炼总结。

## 最后
自己回去翻开历史才更有感慨，技术的演进脱离不了行业的发展，或者说的更直接一点，脱离不了『钱』,至少是赚钱方向的影响。
70~90s欧美企业信息化，为了提效降费，那是的都是单体的，而且是特定功能的。
95~00，WWW的出现，出现信息化浪潮，企业信息化再升级，这样有了SOA解决遗留的异构系统。
00~10，PC互联网企业主键成熟，开始改进SOA，改进技术组织结构和体系
10~ ，移动互联网兴起，一大批中小互联网企业的出现，伴随着激烈的竞争，快速的迭代和交付成为微服务出现的背景。
20年后的演进道路我不知道是什么，但它一定是依托历史的背景和时代发展的需要而出现的。

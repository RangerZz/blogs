---
date: 2018-08-29
title: "YYCache 源码收获"
tags:
    - 源码 YY 感悟 收获
categories:
    - 源码
comment: true
---
# YYCache 源码收获
## 简介
YYCache是一个高性能的线程安全的缓存框架,分为内存缓存和磁盘缓存两部分,内存缓存通过双向链表实现高速存取,磁盘缓存通过SQLite和文件存储实现高性能存取.

## 收获
* 双向链表(_YYLinkedMap, _YYLinkedMapNode)数据结构的实现和LRU(least-recently-used)缓存淘汰算法的实现.从而衍生的使我明晰了算法复杂度的两个维度,即时间复杂度和空间复杂度的内涵,更对Hash算法产生了浓厚兴趣.
* YYCache的接口都是线程安全的,其中YYMemoryCache通过pthread_mutex(互斥锁)在进行读取操作时加锁解锁来保证线程安全,YYDiskCache通过dispatch_semaphore(信号量)来保证线程安全.从而衍生的使我了解更多关于锁的概念与应用.
* 更底层同时也更高效的CFMutableDictionaryRef的API及使用.
* SQLite的简单使用,未来结合阅读FMDB源码应该收获更多.
* 开发者的设计思路与编程技巧.

## 感悟
工作4年,忙忙碌碌,更多的是在用轮子造应用,伴随着公司的策略,上线运营和放弃运营的应用也写了十多款,工作前两年,写各种类型的应用确实扩宽了知识的广度,但很是缺乏知识的深度,尤其近一年来感觉到深深地技术瓶颈,日常工作更多的是重复之前写过的很多代码,用相同的技术知识点来不同的排列组合.

随着困惑的加深,自学提高迫在眉睫,期间购入了很多书籍,也开始阅读优秀开源项目的源码,讲真,就我近期阅读源码的收获,阅读源码真的是提高自我的有效方式,代码应该是开发者沟通最有效的方式,而阅读源码则会让你产生优秀开发者心灵交流的感觉.

未来我想我会继续阅读更多的优秀源码和书籍,并记录下收获.

## 扩展阅读
[YYCache 源码解析](https://knightsj.github.io/2018/02/03/YYCache%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)

[iOS 开发中的八种锁（Lock）](https://www.jianshu.com/p/8b8a01dd6356)

[关于 @synchronized，这儿比你想知道的还要多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

[正确使用多线程同步锁@synchronized()](https://www.jianshu.com/p/2dc347464188)

[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

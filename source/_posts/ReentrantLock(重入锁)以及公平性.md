---
layout: post
title: ReentrantLock(重入锁)以及公平性
date: 2018-01-03 20:15
categories: java
tag:
	- concurrent
	- synchronize
---

## 简介
ReentrantLock的实现可以替代隐式的synchronized关键字，而且能够提供超过关键字本身的多种功能。
这里有一个锁获取的公平性问题，如果在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁就是公平的，反之，是不公平的，也就是说等待时间最长的线程最有机会获取锁，也可以说锁的获取是有序的。ReentrantLock提供了一个构造函数，能够控制这个锁是否公平。而锁的名字也是说明了这个锁具备重复进入的可能，也就是说能让当前线程多次的进行对锁的获取操作，这样的最大次数是限制是Integer.MAX_VALUE,约21次左右。
事实上公平的锁机制没有非公平的效率高，因为公平的获取锁没有考虑到操作系统对线程的调度因素，这样造成JVM对于等待中的线程调度次序和操作系统对线程的调度之间的不匹配。对于锁的快速且重复获取过程中，连续获取的概率是非常高的，而公平锁会压制这种情况，虽然公平性得以保障，但是响应比却下降了。



<!-- more -->



## 实现分析
代码结构如下：
![代码结构](/assets/blogImg/reentranlock-img01.png)

ReentrantLock持有一个Sync对象,该对象实现了AbstractQueuedSynchronizer，而Sync有两个具化实现类FairSync和NonfairSync。
Sync(AbstractQueueSynchronizer)中持有两个Node对象tail和head来维持一张等待线程的链表。

FairSync和NonfairSync的具体区别点在于：
1. lock的时候,FairSync直接调用的acquire(1),当前线程直接进入等待链表中，而NonfairSync则是优先通过compareAndSetState尝试去获取一次锁(修改当前锁的状态)，如果成功则直接拿到锁，不成功则调用acquire(1)进入等待队列。
2. tryAcquire时候，FairSync先通过判断前面没有等待线程的时候在进行尝试获取锁，而NonfairSync直接就尝试获取。FairSync执行的顺序完全安装等待队列中的顺序执行，而NonfairSync则是打破这种规则先直接获取，获取的到就可以，获取不到才进入等待队列。


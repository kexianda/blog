---
title: '[并发系列-3] 从AQS到futex(一): AQS和LockSupport'
categories: 技术
tags:
- 并发
date: 2017-08-13
---

ASQ(AbstractQueuedSynchronizer,同步器)是java.util.concurrenct包一个核心基础类, 来用构建锁或其他同步组件. 网上对这个的分析文章非常多,有源代码解读还有画出图示意线程队列的管理([链接](http://blog.csdn.net/javazejian/article/details/75043422)). 可以看Doug Lea的[AQS论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)([中文版](http://ifeve.com/aqs-2/)).

从另外一个角度,继续深入一下:
* (JDK) J.U.C (AQS/LockSupport) =>
* (JVM) HotSpot (Thread/Parker) 线程和block/unblock实现 =>
* (C runtime) 到glibc中的pthread(NPTL)的实现 =>
* (Kernel) Linux内核的线程和futex 的实现

从AQS到HotSpot, 再到glibc, 最后到内核的futex. 纵向一条线深入下来, 粗略地了解下各个层次的实现, 做个笔记.
<!--more-->
## AQS简述
同步器的语义其实很直观, 就是内部有个原子性的状态变量(锁变量), 同步器实现线程队列的管理和线程的block/unblock.
* 同步状态(原子性)；
* 线程的阻塞与解除阻塞(block/unblock)；
* 队列的管理；

同步器的语义或API, 最核心的语义(API)就两个acquire()/release().  
acquire():
* 检查状态变量, 如果不允许, 把线程进入队列, block当前线程
* 如果允许, 设置状态变量表示占有

release():
* 更新状态变量
* unblock一个或多个等待队列的线程. // LockSupport.unpark(thread);

在J.U.C的实现中, AQS有两个队列, 一个是同步队列,线程被阻塞了,等待获取锁. 另一个是条件队列. 队列节点是内部类Node. 内部类ConditionObject用来构建条件队列，当Condition.wait()后，线程将会加入条件队列中; Condition.signal()后，线程将从条件队列转移动同步队列中进行锁竞争。

在具体实现中, AQS不仅是简单的acquire和release,会更精细的API, 比如:  
* 独占/共享模式的 acquire/acquireShared和release/releaseShared  
所谓独占共享模式,得看具体的逻辑, 比如,CountDownLatch和ReentrantReadWriteLock的独占分享锁的逻辑不一样, 所以AQS需要子类自己去定义.
* 阻塞式非阻塞式, acquire/tryAcquire  
根据状态判断,如果不允许, acquire是block线程, tryAcquire是返回.
*  timed mode.
加上时间限制, 最底层是操作系统根据时间限制唤醒线程.

## 同步状态的操作
状态在AQS里是个:
```java
volatile int state;
```
volatile保证可见性(内存屏障里的acquire/release语义), 32位方便原子操作.
compareAndSet是用unsafe.compareAndSwapInt实现.
在x86平台上是用 cmpxchg 指令实现, 在多核CPU还要加个lock锁下总线(老CPU), 现代的CPU上lock前缀是锁cache line.

state的语义是子类按自己的逻辑定义, 可以当中一个简单布尔类型,或者不同的bit不同语义.比如, 在ReentrantReadWriteLock的精巧设计中,state第16位的bit用是标识EXCLUSIVE状态.

## 阻塞(block/unblock)的实现(LockSupport)
LockSupport主要干两件事情:
* park/unpack(也就是block/unblock线程), 这个通过native实现, 在JVM里支持
* 给JDK的Thread类设置 blocker, 用于监控和统计

park()是阻塞当前线程.  
unpark(thread), 指定参数, 明确唤醒哪个线程.
这两个操作是native, 在JVM中实现.

### 其他
Doug Lea的[AQS论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)里有段话:
>Method LockSupport.park blocks the current thread unless or until a LockSupport.unpark has been issued. (Spurious wakeups are also permitted.) Calls to unpark are not "counted", so multiple unparks before a park only unblock a single park.
Additionally, this applies per-thread, not per-synchronizer. A thread invoking park on a new synchronizer might return immediately because of a "leftover" unpark from a previous usage. However, in the absence of an unpark, its next invocation will block.

非常拗口, 看得人头晕, 看了代码, 其实很简单.
JVM层实现时, parker有个counter字段, LockSupport.unpack(thread)时把counter=1. 重复调用N次也好,counter还是1. unpack调用多次也就是一次的效果.
LockSupport()(当前线程)时, 如果counter>0(逻辑上相当于: 当前线程准备park, 发现别人已经准备唤醒我了,那么我就不阻塞了), 置0,然后返回了.


## 同步队列 / 条件队列的管理
需要注意一点: J.U.C的实现是自己管理线程的队列, 而不是依赖底层的锁(比如glibc的mutex/condition)来管理线程队列.
具体的实现逻辑比较复杂, 网上源码分析极多, 不赘述.

JVM,glibc,linux层的笔记待续...

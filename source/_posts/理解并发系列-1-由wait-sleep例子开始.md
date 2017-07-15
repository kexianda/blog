title: 理解并发系列(1) 由wait/sleep例子开始
categories:
- 技术
tags:
  - 并发
date: 2017-07-10 21:45:57
---

## 0. 引子
实习生同事小T和我讨论, Java里的wait和sleep区别, 他网上一查, 答案大概都是:
* sleep 不会释放锁; 而wait 会释放锁
* wait要先synchronized

```java
//代码片段
sychronized(this) {
  Thread.sleep();
  //this.wait()
}
```
打破砂锅问到底的小T不满意答案,
* 啥叫释放锁持有锁? 啥叫阻塞队列
* 线程怎么放弃或拿到CPU的时间切片  
<!--more-->

## 1. 简单问题: 啥叫持有锁,释放锁,阻塞队列? 啥叫锁?

### 非常粗糙的说,
所谓锁是: 某个flag(某字段) + 用来管理线程的对列.   
A线程持有锁的过程, A线程发现flag没人标记, 那它来标记好, flag你现在是大A我的人了, 别人看到会自己滚开, 大A线程可以继续欢快的干活了.
阻塞队列: 比如说,小B线程恰好也来查看一下flag, 发现flag已有归宿, 那么小B就无奈的跑到某个线程队列里, 且告诉操作系统,我暂时不跑CPU了,默默在队列里等待机会.  
释放锁: 意思是, 大A线程干完活, 把flag自己留下标记擦干净. 留给别人来抢了. 要不要放啥队列,看需求
所谓继续持有锁, 在上面的代码例子就是, sleep后, flag里继续保持自己的标记, 别人过来,自动闪开.


### 稍微稍微详细精确一点儿说,  
Java的synchronized会编译成monitorenter/monitorexit两条bytecode. 具体的语义, JVM来翻译实现.

这里synchronized的锁, 是个monitor(在HotSpot是ObjectMonitor类), monitor里有个owner字段, 表示谁拥有它, monitor里有线程队列(更准确的讲有两个_EntryList和_WaitSet).   
A线程设置owner字段为自己, 持有了锁.   
B过来后看到owner已经被A线程占有了,默默的去_EntryList里等着, 让操作系统阻塞自己.  
A线程活没干完但缺少啥东西了, 自己在这个monitor上wait()一把, 进入_WaitSet队列, 让操作系统阻塞自己, 把owner字段设置为null, 释放了锁. 或者是A干完活, 执行monitorexit, 把owner字段清掉, 释放锁.

为什么整两个队列_EntryList和_WaitSet?  因为Object.notify()的时候, 把_WaitSet里线程放到_EntryList里做好下一次抢占锁的准备.

## 2. 不简单问题: 为何wait要先sychronized?
否则会抛出IllegalMonitorStateException异常.
为什么要这样, 我个人理解是觉得是语义使然. wait和notify搭配使用, 比如生产者消费者情景, 消费者需要等待,那么wait, 放弃锁.  生产可以拿到锁生产数据然后notify消费者.  如果wait不放弃锁, 那么后面的生产拿不到锁,更无法notify了.
可以参考[这篇文章](http://www.xyzws.com/javafaq/why-wait-notify-notifyall-must-be-called-inside-a-synchronized-method-block/127)

Java里的wait/notify和Condition Variable类似, 会写一篇文章继续深入讨论Condition Variable.


## 3. JVM里怎么实现wait的语义

ObjectMonitor对象中有两个队列：
_WaitSet 和 _EntryList. _owner指向获得ObjectMonitor对象的线程. 线程试图取得锁(monitorenter)但被占有, 那么被放到WaitSet里, 线程自己wait了, 放入_EntryList里去.  
这里引用HotSpot(OpenJDK 9)的代码, 不是要详述JVM的实现(太太复杂了), 我极度简化, 仅仅留下几行最核心的逻辑. 我添加注释.

```c++
ObjectMonitor::wait() {

  // 省略若干代码...

  //把线程用ObjectWaiter包装起来, 准备放入_WaiterList里
  ObjectWaiter node(Self);
  node.TState = ObjectWaiter::TS_WAIT;
  Self->_ParkEvent->reset();     //
  OrderAccess::fence();          // ST into Event; membar ; LD interrupted-flag

  // ...

  //把线程放入_WaiterList里, 这个过程需要好几步骤,不能被打断,
  //也需保护起来, 用自旋锁啥的保护起来.  
  Thread::SpinAcquire(&_WaitSetLock, "WaitSet - add");
  AddWaiter(&node);
  Thread::SpinRelease(&_WaitSetLock);

  // ...

  // 释放锁, 省略若干代码...
  exit(true, Self);     // exit the monitor

  // ...

  //这时候才是真的调用系统调用, 休眠自己
  // 实现其实是用pthread的API, pthread_cond_wait等接口
  // 而pthread_cond_wait又是调用Linux系统调用futex
  Self->_ParkEvent->park();

  // ...
}

ObjectMonitor::exit() {
  // ...

  // 把owner清空,
  // 这里OrderAccess的语义和必要性,以后再聊
  OrderAccess::release_store_ptr(&_owner, NULL);   // drop the lock
  OrderAccess::storeload();
  // ...
}
```

## 4. sleep的语义
语义比较简单, 猜测实现大约就是扔到某队列(可以interrupted后可唤醒), 然后系统调用,让内核休眠自己.
赖得去看HotSpot实现细节, 太繁复了, 有空再看.


## 5. 概要回答下"线程怎么放弃或拿到CPU的时间切片"
Java程序员调用wait/sleep后, JVM里做一大堆处理, JVM需要在用户空间维护各种线程队列和状态的字段来实现java的语义.
最后JVM通过glibc这层, 系统调用(通常是futex), 这时候, system_enter指令, 进入内核空间, 运行内核代码, Linux调度其他线程干活, 通过switch_to()函数来实现, 保存寄存器上下文什么的, 改变pc值(x86里rip寄存器), 这时候CPU就指向了其他线程的代码块了. 被休眠的进程已经让出CPU, 新的进程拿到了CPU.
这是非常概要的回答. 细节得看内核源码.


## 6. 还有很多有趣问题
* HotSpot代码这句OrderAccess::release_store_ptr(&_owner, NULL), release_store语义是什么?
* JVM实现代码的里的中内存屏障,OrderAccess::storeload()等, 啥意思,干啥的. 涉及到内存模型.
* synchronized, 其实很复杂, 有偏向锁优化, 锁膨胀等.
* JVM调用系统,休眠线程涉及到glibc的NPTL实现,和linux内核的futex. 这是彻底理解线程在用户空间和操作系统内核空间怎么运作的的基础.

希望能有空, 以后可以继续写.

---
title: '[并发系列-2] 为什么要有Condition Variable?'
categories:
 - 技术
tags:
 - 并发
date: 2017-07-15 12:26:58
---
儿子最近不在家, 我开机时也没人过来胡乱锤打我的键盘了. 有空来写篇技术文章, 不是宏大的分布式大数据深度学习啥, 深入扣个细节, 就像孔乙己写"回"字, 回囘囬廻...

Condition Variable是个同步原语(synchronization primitives), 用来协调不同线程的逻辑顺序. 和实习生同事小T讨论时, 谈到了两个问题:
1. 有了mutex, 为啥还要整个Condition variable的概念出来?
2. Condition为什么要跟一个锁(mutex)一起用? 比如, pthread_cond_wait(cond, mutex), 再比如Java里的condition是由锁newCondion()生成.

我觉得这两问题很有深度, 小结一下我个人的理解.
<!--more-->

## 为什么要有Condition variable概念?
mutex, 就是线程们一起竞争锁, 谁拿到谁先跑. 用来保护Critical section. 比如独木桥只能过一个人, 多个人跑到河边, 竞争独木桥, 一次上一个人, 过完河再开闸过下一个. 虽然有竞争策略, 但是没有规定A一定要在B前面先过. 没有具体的次序关系.

Condition, 就是线程们干活前先看看, 是否满足开始干活的条件, 不满足则让系统休眠自己.
生产者消费者例子是个典型. 消费者需要等到有数据, 消费者需要等到空间.
再比如接立赛上, 第二棒B跑道上站好准备了,但是不能跑, 没有拿到第一棒传来的交接棒, 于是等待(cond_wait), 第一棒A跑完某100米(条件满足了), 把交接棒给第二棒B(A发个signal). 第二棒B拿到了交接棒(wake up)接下去就开跑.  
这种同步(次序的协调关系)用单纯的互斥锁实现很费劲. 用Condition variable, 对用程序员而言, 直观了很多. 这是对问题(1)的回答.

## 先看Condition variable怎么用

### C/C++/Java的例子
pthread标准里是pthread_cond_t/pthread_cond_wait()/pthread_cond_signal, Java里比如ReentrantLock.newCondion(). C++是std::condition_variable.
Windowns上也有类似的API. 通过代码可以看到, Condition都是绑在一个锁上.
```c
// C, pthread API
pthread_mutex_lock(mutex_lock);
while(buffer.size == 0){  
   pthread_cond_wait(condition_variable, mutex_lock);
}
//...
pthread_mutex_unlock(mutex_lock);
//...
```

```c++
// C++ 11.
std::unique_lock<std::mutex> lk(mutex);
cond.wait(lk, []{return ready;});  //Equivalent to: while (!pred()) wait(lock);
//...

//cond.notify_one();
```

```java
// Java
Lock lock = new ReentrantLock();  
Condition condition = lock.newCondition();   

lock.lock();  
while(flag == true) {                
    condition.await();
}          
//...
```
看代码示例, 除了Condition需要mutex配合问题, 问题(2)没回答, 还带来一个新的问题(3): ** condition variable wait都有个while loop, 为什么? **


## Condition wait的实现
看看实现, 有利于理解上面的问题.
Java的同步原语Condition在linux平台上HotSpot中是用pthread实现的, C++标准库也只是定义接口, 在linux平台也是pthread/NPTL实现的.
原理都是一回事. 所以,这里仅以glibc/NPTL为例来看实现和讨论为什么.[知乎里也有类似讨论][1],可以看看.
我看了下glibc的代码,谈谈我的理解.
### glibc wait的简单流程图:
蓝色框里就是pthread_cond_wait的简化逻辑. 里面调用了linux的系统调用futex_wait,把休眠自己交出CPU, 这个也有意思,可以深入了解下, 不过这里暂且略过.

![](http://ot49rzljt.bkt.clouddn.com/image/tech/pthread_cond_wait.png)

### glibc/JDK中的实现
简化了逻辑，暂且只关心最核心的基本逻辑.
```c
// https://github.com/lattera/glibc/blob/master/nptl/pthread_cond_wait.c
int
__pthread_cond_wait (pthread_cond_t *cond;
     pthread_mutex_t *mutex)
{
   // 先释放mutex, 为什么?
   // 因为要和他人,合作, 我睡眠了, 别人可以进去拿锁然后改变条件
   err = __pthread_mutex_unlock_usercnt (mutex, 0);
   do {
     // 我们准备在cond_cont->__data.__futex这个变量上 休眠自己了

     // 关键点， 有竞争
     // futex_val 是目前线程观察到值，
     unsigned int futex_val = cond->__data.__futex;

     // Prepare to wait.  Release the condvar futex.  
     lll_unlock (cond->__data.__lock, pshared);

     // Wait until woken by signal or broadcast.  
     // 所以用户态应该将自己看到的*uaddr的值作为第二个参数传递进去，
     // futex_wait真正将进程挂起之前一定得检查lockval是否发生了变化，
     // 并且检查过程跟进程挂起的过程得放在同一个临界区中。
     lll_futex_wait (&cond->__data.__futex, futex_val, pshared);
   } while(val == seq || cond->__data.__woken_seq == val);
   // why loop, 如果醒了, 但是woken_seq里并没有变化, 那么继续futex_wait的逻辑

   // 如果醒来， 立马拿mutex 锁
   // Get the mutex before returning.  
   return __pthread_mutex_cond_lock (mutex);
}

```
熟悉JDK的话, 发现这个逻辑和J.U.C.AbstractQueuedSynchronizer.ConditionObject.await()很类似.
```Java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

## 回答前面的问题
根据我读glibc的个人理解,
### Condition variable需要mutex来保护条件变量
条件变量的判断过程不能有data racing.
不能发生这种情况:
* A线程里条件刚刚判断好了, 需要wait, 刚准备去加入wait queue让系统休眠自己; 就在这个间隙,被OS切换出去了.
* 另一个B线程导致条件变化了(A其实不应该wait了), B发出signal, 因为这个时候wait queue里还没有A线程(因为A还没成功加进去呢), signal也是浪费表情,浪费掉了.
* 然后A呢切回继续运行,准备加入condition关联的等待队列休眠自己. 然后就可能醒不过来了, 因为B不会发signal给A了.

流程图里的(1)mutex.lock和(3)mutex.unlock保护了 (2) pred()期间, 条件不要改动.
这是对问题(2)的回答.

### 为什么Condition variable wait都有个while loop?

根据上面的流程图讨论, 看下面这种情况:
* A1线程在流程图(4)里wait着,  
* B线程完成它的任务, 把条件改了, pred()==flase了,发个signal到condition,
* A1从(4)醒过来, 但还没开始(5)
* A2线程发现条件改了, pred()==false, 不需要休眠了, 直接抢活干(6)working, 把条件又改了, 注意这时候又pred()==true了. (7)也跑完.
* A1如果不再次检查 pred, 是否需要睡眠, 就会在条件不满足的情况下去干(6), 而(6)必须在pred()==false,才能做. 显然出问题了.  

这就是为什么要加个while loop. 回答了问题(3).
[update: 有个术语spurious wakeup描述这种情况]

## 小结
Condtion Variable是个比mutex稍复杂的原语, 这个抽象了一层的概念, 给程序员带来方便. 典型应用场景有生产者消费者.  
有篇[介绍文章](http://pages.cs.wisc.edu/~remzi/OSTEP/threads-cv.pdf), 里面的参考论文文献, 有空值得一看.

### 顺便一提,
从接口上看, C++里提供重载函数, 把条件判断作为lambda传进去. 把这个while loop的必要性隐藏到接口里面去, 而不是让程序来处理. C++接口设计最优雅.
```c++
cond.wait(mutex, []{return pred();});
```


[1]: https://www.zhihu.com/question/24116967 "知乎里讨论pthread_cond_wait为什么要传mutex"
[2]: "https://www.quora.com/What-is-the-difference-between-mutex-condition-variable-semaphore-and-monitor" "Quora的一个讨论"

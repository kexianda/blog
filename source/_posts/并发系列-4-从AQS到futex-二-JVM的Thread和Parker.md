---
title: '[并发系列-4] 从AQS到futex(二): HotSpot的JavaThread和Parker'
categories: 技术
tags:
- 并发
- HotSpot
date: 2017-08-16
---

J.U.C.中AQS管理线程队列, LockSupport用来block/unblock线程. 通过HotSpot(Java 9)的源码来粗略的看看JVM这层的实现:
* HotSpot的Thread/JavaThread类的简介
* HotSpot的Parker的实现细节
<!--more-->

## HotSpot的Thread/JavaThread
HotSpot里的Thread类对应着一个OS的Thread, JavaThread类继承字Thread, JavaThread实例对应着一个Java层的Thread.
所以, 在Java层的Thread, 在操作系统上对应一个thread, linux上就是一个轻量级task.

```c++
// thread.hpp
class Thread {

  // OS data associated with the thread
  OSThread* _osthread;  // Platform-specific thread information

  ParkEvent * _ParkEvent;    // for synchronized()
  ParkEvent * _SleepEvent;   // for Thread.sleep

  // JSR166 per-thread parker
  Parker*    _parker;
  // 略...
};

class JavaThread: public Thread {
  // 指向Java Thread实例,
  //oop是HotSpot里指向一个Java level的实例, 一个gc对象.
  oop            _threadObj; // The Java level thread ,
  JavaFrameAnchor _anchor;   // Encapsulation of current java frame and it state
  CompiledMethod*   _deopt_nmethod; // CompiledMethod that is currently being deoptimized

  //
  volatile JavaThreadState _thread_state;

  // 非常多,略...
};
```
Thread类里有两个ParkEvent和一个Parker, 其实ParkEvent和Parker实现和功能都类似,只是源码没有重构而已. 一个是给实现synchronized关键字用的, 一个是给Thread.sleep/wait用的. parker是用来实现J.U.C的park/unpark(阻塞/唤醒).
```
// A word of caution: The JVM uses 2 very similar constructs:
// 1. ParkEvent are used for Java-level "monitor" synchronization.
// 2. Parkers are used by JSR166-JUC park-unpark.
```
JavaThread和Java的线程一一对应, 成员变量oop _threadObj指向Java层的thread对象.
OSThread是通过os::create_thread()创建, 最后还是调用POSIX phtread, glibc在linux平台上就是fork一个轻量级task.
```c++
//thread.cpp
os::create_thread(this, thr_type, stack_sz);

//linux_os.cpp
pthread_t tid;
int ret = pthread_create(&tid, &attr, (void* (*)(void*)) thread_native_entry, thread);
```

## HotSpot的Parker

LockSupport是调用Unsafe_Park/Unsafe_unpark, unsafe只是调用parker.park()(另,还收集trace/profile信息). 看看Parker:

```c++
//park.hpp
class Parker : public os::PlatformParker { /*略...*/ };

//os_linux.hpp
class PlatformParker {
 pthread_mutex_t _mutex[1];

 //一个是给park用, 另一个是给parkUntil用
 pthread_cond_t  _cond[2]; // one for relative times and one for abs.
 //略...
};
```
常见用pthread写多线程程序, 一般来说是多个线程竞争mutex/cond变量, 把线程队列的管理的逻辑扔给mutex/cond去做, 底层的glibc或linux kernel做线程队列的管理. 而在HotSpot里, 每个JavaThread实例里都自己的pthread mutex/cond变量.  

那么问题是:
1. 为什么这么整?
2. 每个线程有自己的mutex, 那哪来的有竞争呢?

1, JDK中的LockSupport只是用来block(park,阻塞)/unblock(unpark,唤醒)线程, 线程队列的管理是JDK的AQS处理的. 从Java层来看, 只需要mutex/cond提供个操作系统的block/unblock API即可.

2, 但是, 实现时, 竞争还是有的, 比如当前线程某thread_a.park()时, 而另外一个线程thread_b.unpark(thread_a), 这时候, 对于thread_a的mutex来说有竞争的.
只不过, 对于JDK而言, 可以透明,只是需要关注block/unblock是阻塞/唤醒了线程操作.

不同系统实现不一样, 从Parker::park的代码可以看到, 这里的data racing主要是处理: 如果别的线程在unpark它的情况.
只看Linux的(已经删掉一些不是核心逻辑的代码, 我加了自己的理解,写在注释里):
```c++
void Parker::park(bool isAbsolute, jlong time) {

  //  如果别的线程已经unblock了我.  
  //  这里并没有拿到mutex的锁, 需要Atomic::xchg和barrier保证lock-free代码的正确.
  // We depend on Atomic::xchg() having full barrier semantics
  // since we are doing a lock-free update to _counter.
  if (Atomic::xchg(0, &_counter) > 0) return;

  // safepoint region相关, 我对细节不详.
  // safepoint region大致的了解, 见RednaxelaFX的回答https://www.zhihu.com/question/29268019
  ThreadBlockInVM tbivm(jt);

  // 如果别的线程正在unblock我, 而持有了mutex, 我先返回了,没有必要在_mutex上等
  if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
    return;
  }

  // 如果别的线程已经unblock了我, no wait needed
  // 已经拿到了mutex, 所以不需要和前面一样Atomic::xchg了.
  int status;
  if (_counter > 0)  {
    _counter = 0;
    status = pthread_mutex_unlock(_mutex);
    OrderAccess::fence();
    return;
  }

  // 记录线程的状态
  OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
  jt->set_suspend_equivalent();
  // cleared by handle_special_suspend_equivalent_condition() or java_suspend_self()

  // 这一坨, 就是block自己这个线程了.(Java层当前执行的线程)
  if (time == 0) {
    _cur_index = REL_INDEX; // arbitrary choice when not timed
    status = pthread_cond_wait(&_cond[_cur_index], _mutex);
  } else {
    _cur_index = isAbsolute ? ABS_INDEX : REL_INDEX;
    status = pthread_cond_timedwait(&_cond[_cur_index], _mutex, &absTime);
  }
  _cur_index = -1;

  // 已经从block住状态中恢复返回了, 把_counter设0.
  _counter = 0;
  status = pthread_mutex_unlock(_mutex);

  // 要保证多线程的正确性要十二分小心
  // 这里的memory fence 是一个lock addl 指令, 加上compiler_barrier
  // 保证_counter = 0 是对调用unlock线程是可见的.
  // Paranoia to ensure our locked and lock-free paths interact
  // correctly with each other and Java-level accesses.
  OrderAccess::fence();

  // 已经醒过来, 但如果有别人在suspend我,那么继续suspend自己.
  // If externally suspended while waiting, re-suspend
  if (jt->handle_special_suspend_equivalent_condition()) {
    jt->java_suspend_self();
  }
}
```

再看unpark, B线程唤醒A线程, 是B调用了A的parker::unpark()(搜Unsafe_Unpark的代码).
```c++
void Parker::unpark() {
  int status = pthread_mutex_lock(_mutex);
  const int s = _counter;
  _counter = 1;
  // must capture correct index before unlocking
  int index = _cur_index;
  status = pthread_mutex_unlock(_mutex);
  if (s < 1 && index != -1) {
    // thread is definitely parked
    status = pthread_cond_signal(&_cond[index]);
  }
}
```
Parker::unpark()比较简单, 设置_counter=1, 并unlock mutex和cond_signal.
_counter不会增加, 设死为1. 这就是Doug Lea的[AQS论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)里说'Calls to unpark are not "counted".

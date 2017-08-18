---
title: '[并发系列-5] 从AQS到futex(三): glibc(NPTL)的mutex/cond实现'
categories: 技术
tags:
- 并发
- glibc
date: 2017-08-17
---

[上篇笔记](http://kexianda.info/2017/08/16/%E5%B9%B6%E5%8F%91%E7%B3%BB%E5%88%97-4-%E4%BB%8EAQS%E5%88%B0futex-%E4%BA%8C-JVM%E7%9A%84Thread%E5%92%8CParker/)大致看了Java同步互斥机制在JVM层的实现. HotSpot在Linux平台是用pthread实现. 这篇继续对glibc(NPTL)在Linux平台上如何实现mutex和condition做个笔记.  
glibc的源码极其难读, 因为要处理平台区别,性能优化,特殊情况处理等等, 源码的可读性不在大神们的考虑之列. 这里不是逐行源码解读, 列出来的代码都不是glibc的原版代码, 改动和简化了, 类伪码了,方便理解, 也把我的理解作为注释放在代码里.  
* pthread_mutex_t和lock/unlock操作,
* low level lock的汇编实现, 最为tricky的地方
* pthread_cond_t和wait/signal.
<!--more-->

## pthread mutex
### pthread_mutex_t
```c
pthread_mutex_t {
	int __lock; // 锁变量, 传给系统调用futex,用作用户空间的锁变量
	usigned int __count;  // 可重入的计数
	int __owner;   // 被哪个线程占有了

	int __kind;  // 是否进程间共享,等等...
  // int __nusers; // 其他字段略
}
```
pthread mutex可设置属性, 有如下类型(不同类型的mutex的lock/unlock实现不同):
* PTHREAD_MUTEX_TIMED_NP，这是缺省值，也就是普通锁。
* PTHREAD_MUTEX_RECURSIVE_NP，可重入
* 其他的先略了...

### pthread_mutex_lock
pthread_mutex_lock调用LLL_UNLOCK(基于Linux的futex), 去拿到锁或阻塞自己. 另外,对可重入的锁进行计数.   
(不过, JDK/JVM层有自己的可重入锁设计, 并没有用到 PTHREAD_MUTEX_RECURSIVE_NP 的mutex.)
```c
pthread_mutex_lock (pthread_mutex_t *mutex) {
  if (type == PTHREAD_MUTEX_TIMED_NP)) {
      /* Normal mutex.  */
      /*LLL_UNLOCK宏是lll_unlock (mutex->__data.__lock, PTHREAD_MUTEX_PSHARED (mutex));
       PTHREAD_MUTEX_PSHARED 是不同进程间的, 线程见的话,为false
      */
      LLL_UNLOCK(mutex);
  }
  else if (type == PTHREAD_MUTEX_RECURSIVE_NP) {
      /* Recursive mutex.  */
      pid_t id = THREAD_GETMEM (THREAD_SELF, tid);

    /* 若已经持有了此锁, 增加计数, 无需block此线程 */
    if (mutex->__data.__owner == id){
			  ++mutex->__data.__count;
			  return 0;
		}
		// 去判断锁变量, 如果不行, 被OS休眠掉
		LLL_MUTEX_LOCK (mutex);    
		// 拿到了锁, 锁变量是ok的,则设置count
		mutex->__data.__count = 1;
	}
	// ...特殊处理和其他类型锁的逻辑忽略...
}
```
### pthread_mutex_unlock
```c
pthread_mutex_unlock (pthread_mutex_t *mutex) {
	if (type == PTHREAD_MUTEX_TIMED_NP) {
		mutex->__data.__owner = 0;
		lll_unlock (mutex->__data.__lock, PTHREAD_MUTEX_PSHARED (mutex));
		return 0;
  }
	else {
		// if (type == PTHREAD_MUTEX_RECURSIVE_NP) ...
		// 省略不看了 ...
	}
}
```
粗略来看, mutex主要是调用底层的lll_lock/lll_unlock, 其实就是调用futex的FUTEX_WAIT/FUTEX_WAKE操作, 来实现线程的休眠和唤醒工作.


## low level lock
lll_lock/lll_unlock的lll应该是low level lock的缩写了. 这部分最为tricky, 不是因为汇编, 是并发的处理.
lll_lock, 先原子性的检查uaddr中计数器的值是否为val,如果是则让进程休眠，直到FUTEX_WAKE或者超时(time-out)。也就是把进程挂到uaddr相对应的等待队列上去。看代码:
```c
__lll_lock_wait_private (int *futex)
{
  do {
    int oldval = atomic_compare_and_exchange_val_24_acq (futex, 2, 1);
    if (oldval != 0)  // 步骤1 用户空间的代码
    lll_futex_wait (futex, 2, LLL_PRIVATE);//步骤2, 马上陷入内核了
  }
  while (atomic_compare_and_exchange_val_24_acq (futex, 2, 0) != 0);
}
```
do-while循环在这里的作用是: 从内核空间的wait里刚出来, 检查锁变量是不是就在这会儿被别人抢走, 如果以及被别人抢走, wait里出来继续后面的逻辑, 就要出错了.

这里还有个非常重要的点提一下:  **当前线程(用户空间)看到锁变量被设置了, 准备进入内核, 打算去休眠自己, 如果在这个步骤1和步骤2的空隙里, 锁变量\_\_lock又被清掉了, 这时候, 当前线程其实没有必要休眠了. 如果内核真的把当前线程休眠了,那就出问题了. 内核的futex的实现了还会再次判断\_\_lock变量的值.**, 保证不出问题. 这个需要详细讨论futex了, 下次继续.

继续看lll_futex_wait, 就是汇编进行系统调用陷入内核.
```asm
  ;  /*int futex (*uaddr, op, val, timeout,*uaddr2, val3);*/
	; gdb> break __lll_lock_wait (看源码不如gdb看汇编)
  pushq	%r10
	pushq	%rdx
	; 设置futex系统调用的参数, 比如FUTEX_WAIT/FUTEX_WAKE操作
	xorq	%r10, %r10
	movl	$2, %edx  
	xor    %r10,%r10   ; /* No timeout.  */
  mov    $0x2,%edx
  xor    $0x80,%esi
  cmp    %edx,%eax
  jne    __lll_lock_wait+29
  nop
  mov    $0xca,%eax  ; futex系统调用号,
  syscall            ; syscall指令陷入内核了, 调用内核的futex.
```
lll_unlock也类似,只不过传的是FUTEX_WAKE参数.
终于看到**陷入内核的代码**了. (我的好奇心基本上被满足了...)

## pthread condition variable
POSIX pthread标准里是pthread_cond_t/pthread_cond_wait()/pthread_cond_signal.
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
     // futex_val是目前线程观察到值，
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
熟悉JDK的话, 发现这个逻辑和 J.U.C的AbstractQueuedSynchronizer.ConditionObject.await()很类似.
![](http://ot49rzljt.bkt.clouddn.com/image/tech/pthread_cond_wait.png)  
蓝色框里就是pthread_cond_wait的简化逻辑. 里面调用了FUTEX_WAIT,前面已经详述.
这里tricky的地方类似, 也是do-while循环, 从wait中醒来, 需要再次检查.

pthread_cond_signal,略...

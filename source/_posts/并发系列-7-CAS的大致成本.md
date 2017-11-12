---
title: '[并发系列-7] CAS的大致成本'
categories: 技术
tags:
- 并发
date: 2017-11-12
---
多线程的高并发总是个很tricky的事情. 为了避免操作系统的线程context switch成本(以千个ns为单位了), lock-free编程中用各种原子操作(CAS等和内存屏障). 极端的地方, 甚至CAS操作的成本都要考虑. 比如Java中的synchronized的偏向锁/轻量级锁优化就考虑减少CAS操作. 还有JUC包中的一些实现也是尽量减少CAS.
那么CAS大致上什么样的一个成本呢?
<!--more-->
## Java ConcurrentLinkedQueue的进队列例子
先看ConcurrentLinkedQueue(Java 8)的进队列例子,看看Doug Lea大神对CAS怎样考虑tradeoff, 对CAS的成本有个大致印象.
```Java
/**
     * Inserts the specified element at the tail of this queue.
     * As the queue is unbounded, this method will never return {@code false}.
     *
     * @return {@code true} (as specified by {@link Queue#offer})
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        checkNotNull(e);
        final Node<E> newNode = new Node<E>(e);

        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            if (q == null) {
                // p is last node
                if (p.casNext(null, newNode)) {
                    // Successful CAS is the linearization point
                    // for e to become an element of this queue,
                    // and for newNode to become "live".
                    if (p != t) // hop two nodes at a time
                        casTail(t, newNode);  // Failure is OK.
                    return true;
                }
                // Lost CAS race to another thread; re-read next
            }
            else if (p == q)
                // We have fallen off list.  If tail is unchanged, it
                // will also be off-list, in which case we need to
                // jump to head, from which all live nodes are always
                // reachable.  Else the new tail is a better bet.
                p = (t != (t = tail)) ? t : head;
            else
                // Check for tail updates after two hops.
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
```
代码弄的那么晦涩, 付出多跑一次循环的成本, 就是尽量减少CAS操作,不是每次进队列都更新tail变量.

## 成本分析
CAS在x86下是lock cmpxchg指令, 成本有:
* CPU cache一致性(MESI)的同步成本.
* CPU cache miss
更重的成本在于, cmpxchg一次可能的load cache miss和一次可能的store cache miss, 在多核高并发的环境下, 基础类库的实现中, 竞争激烈, CAS导致的cache miss就高了. 访问memory的话,成本到100+ns级别.

没有写driver亲自测试, CAS成本粗糙的估计:
非竞态: 大致十几个ns.
激烈竞争状态: 1~2百个ns.

## 一个玩脱了的例子
Google的gperftool(TCmalloc)中的SpinLock(Author:大神Sanjay Ghemawat)曾有个[bug](https://github.com/gperftools/gperftools/issues/494), 为了在非竞争情况下省几个纳秒,不用CAS而用NoBarrier_Load,Release_Store, 不小心玩脱了,导致了高并发下的performance的bug.
```c
inline void Unlock() /*UNLOCK_FUNCTION()*/ {
   uint64 wait_cycles =
       static_cast<uint64>(base::subtle::NoBarrier_Load(&lockword_));
   ANNOTATE_RWLOCK_RELEASED(this, 1);
   base::subtle::Release_Store(&lockword_, kSpinLockFree);
   if (wait_cycles != kSpinLockHeld) {
     // Collect contentionz profile info, and speed the wakeup of any waiter.
     // The wait_cycles value indicates how long this thread spent waiting
     // for the lock.
     SlowUnlock(wait_cycles);
   }
 }

```
这个bug其实是个原子性的问题, 脑补下bug发生的条件和导致的性能问题, 很有意思. 后来用CAS操作fix了这个bug.

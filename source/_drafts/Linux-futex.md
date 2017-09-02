---
title: '[并发系列-6] 从AQS到futex(四): Futex/Critical Section简介'
categories: 技术
tags:
- 并发
- glibc
- kernel
date: 2017-08-17
---
Futex是Linux提供的最基础的并发原语, C runtime如glibc的mutex，join，condition variable，semphore都是基于futex实现.
<!--more-->

## Overview(what & why)
[Futex(Fast Userspace muTexes)](https://www.kernel.org/doc/ols/2002/ols2002-pages-479-495.pdf),顾名思义，锁变量是在userspace中. Linux从2.5.7开始支持Futex.
其设计理由基于两点：
* 很多同步是无竞争的，一个线程进入临界区然后退出临界区，这个过程往往并没有线程来竞争。Java的synchronized在JVM中的偏向锁轻量级锁的优化也是基于此理由.
* 陷入内核的成本其实是很高的. 比如Linux系统调用的VSDO优化就是为了避免陷入内核. 之前的系统中，进程间同步机制是对一个内核对象操作来完成，读写锁对象而陷入内核，成本很高。

粗略的来讲，从应用程序角度看，一个线程去拿锁，通过原子的test-and-set(cmpxchg())指令去看锁变量，如果没有其他线程持有此锁变量。那么当前线程设置锁变量，因为锁变量在用户空间，不需要陷入内核，此时内核对此一无所知。 如果下一个线程在前一个线程持有锁变量的时候，也来拿锁，发现已经被别人持有，那么，需要调用futex系统调用传FUTEX_WAIT参数，让系统把我这个线程休眠在futex的锁变量上， 内核会线程休眠，同事建立相关的数据结构(futex_q/futex_queues)关联用户空间的锁变量(uaddr)和线程. 如果锁持有者释放锁的时候，把锁变量设置，并调用sys_futex，传FUTEX_WAKE参数。

释放锁的时候，如果没有其他线程阻塞在锁变量上，其实没有必要调用sys_tutex, 这是个优化，可以看[Futexes Are Tricky](https://cis.temple.edu/~giorgio/cis307/readings/futex.pdf)文中的mutex3实现。在glibc中有个flage FUTEX_WAITERS作用类似。

## 实现(how)
接口：
```c
int futex (int *uaddr, int op, int val, const struct timespec *timeout,int *uaddr2, int val3);
```
常用前四个参数。第三个val传进去的就是futex变量(锁变量)。因为内核需要再次检查val，保证正确性。为什么呢?

```
if (futex_var is true)    // (1)
    futex(uaddr, FUTEX_WAIT, futex_var) (2)
```
在(1)(2)之间，如果futex_var发生了变化,其实不用休眠线程了，如果内核futex实现不再次检查futex_var,把线程休眠可能再也醒不过来了。

在内核中通过一个哈希表来维护所有挂起阻塞在futex变量上的task，不同的futex变量会根据其地址标识计算出一个hash key定位到一个哈希桶链上，阻塞在同一个futex变量的所有task会挂到同一个哈希桶链上：  
(转张图,一图胜千言):
![](https://static.lwn.net/images/ns/kernel/dvh-futexes.png)



## 实践
要写正确的并发库其实是非常困难的，tricky的地方很多。一般来说，应用开发不需要直接用sys_futex. C runtime如glibc里都已经基于futex实现了mutex cond等等(具体实现比较tricky,[这里](http://kexianda.info/2017/08/17/%E5%B9%B6%E5%8F%91%E7%B3%BB%E5%88%97-5-%E4%BB%8EAQS%E5%88%B0futex%E4%B8%89-glibc-NPTL-%E7%9A%84mutex-cond%E5%AE%9E%E7%8E%B0/)有简单介绍)

可以写个小程序，用strace来观察用到的系统调用。
```
strace -c a.out
```

## Windows平台呢?
Windows平台有Mutex和Critical Section，Mutex是kernel object，而Critical Section是user object. 细节可看[critical_section vs mutex](https://blogs.msdn.microsoft.com/ce_base/2007/03/26/critical-section-vs-mutex/).

有人在windows平台做了[micro benchmark](http://preshing.com/20111124/always-use-a-lightweight-mutex/), 二者性能相差40~50倍。

## 参考:
1. [Futex Overview](https://lwn.net/Articles/360699/)  
2. [Always use a lightweight mutex](http://preshing.com/20111124/always-use-a-lightweight-mutex)
3. [Linux Futex浅析] (http://blog.sina.com.cn/s/blog_e59371cc0102v29b.html)

===========================================================================
===========================================================================
2. why we need futex

3. how futex work

http://preshing.com/20111124/always-use-a-lightweight-mutex/

glibc join, mutex, semphore...

http://blog.sina.com.cn/s/blog_e59371cc0102v29b.html
futex_awit_setup(uaddr,val, flags, &q, &hb)

futex_wait_queue_me(hb, &q, to);


https://github.com/torvalds/linux/blob/master/kernel/futex.c#L767
https://lwn.net/Articles/360699/
https://www.akkadia.org/drepper/futex.pdf
http://opensourceforu.com/2013/12/things-know-futexes/
[strace futex](http://blog.csdn.net/jianchaolv/article/details/7544316)
=================================================
## AQS -> LockSupport -> park-> condition

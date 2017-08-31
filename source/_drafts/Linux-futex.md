---
title: Linux futex
tags:
---
1. what is futex
其设计思想其实不难理解，在传统的Unix系统中，System V IPC(inter process communication)，如 semaphores, msgqueues, sockets还有文件锁机制(flock())等进程间同步机制都是对一个内核对象操作来完成的，这个内核对象对要同步的进程都是可见的，其提供了共享的状态信息和原子操作。当进程间要同步的时候必须要通过系统调用(如semop())在内核中完成。可是经研究发现，很多同步是无竞争的，即某个进程进入互斥区，到再从某个互斥区出来这段时间，常常是没有进程也要进这个互斥区或者请求同一同步变量的。但是在这种情况下，这个进程也要陷入内核去看看有没有人和它竞争，退出的时侯还要陷入内核去看看有没有进程等待在同一同步变量上。这些不必要的系统调用(或者说内核陷入)造成了大量的性能开销。为了解决这个问题，Futex就应运而生，Futex是一种用户态和内核态混合的同步机制


Windows Critical Section.
HANDLE mutex = CreateMutex(NULL, FALSE, NULL);
for (int i = 0; i < 1000000; i++)
{
    WaitForSingleObject(mutex, INFINITE);
    ReleaseMutex(mutex);
}
CloseHandle(mutex);

CRITICAL_SECTION critSec;
InitializeCriticalSection(&critSec);
for (int i = 0; i < 1000000; i++)
{
    EnterCriticalSection(&critSec);
    LeaveCriticalSection(&critSec);
}    
DeleteCriticalSection(&critSec);
http://preshing.com/20111124/always-use-a-lightweight-mutex/


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

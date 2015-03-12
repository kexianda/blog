title: Double-checked Locking
tags:
	- C++
	- 多线程
category: 技术
date: 2014/12/28
---

在学习C++11多线程的时候，会碰到一大堆概念，mutex, lock, atomic, memory model, memory barrier， lock-free等。要更好的理解，可以先了解下CPU的Memory Barriers机制(可读Paul McKenny的[Memory Barriers: a Hardware View for Software Hackers][1])，然后看Jeff Preshing的blog, 他的帖子深入浅出，写得非常好。再看看Herb Sutter,Hans Boehm等人的文章。 [Bartosz Milewski的博客][7]也值得看看.

double-checked locking是一个用来学习的好例子。 Scott Meyers和Andrei Alexandrescu两位大牛写过一篇[paper][2]，讨论了double-checked实现的困难(Java的memeory model没有完善之前有同样的问题). Jeff Preshing写篇[文章][3]讨论这个问题.

看完大牛们的文章, 一步一步来动手实现一个Singleton，最后用template泛化.

<!-- more -->
###1. 单线程
Singleton的单线程实现很简单：
```cpp
//header file
class Singleton {
public:
	static Singleton* getInstance ();
private:
	static Singleton* m_Instance;
};

//implementation.  Version 0.1
Singleton* Singleton::m_Instance = nullptr;

Singleton* Singleton::getInstance () {
	if (m_Instance == nullptr) {
		m_Instance = new Singleton;
	}
	return Singleton;
}

```
在单线程环境，这个版本工作的很好。 但在多线程线环境下有data race了， 关键在那个if判断， 多个线程可能同时进入if里面。

###2.多线程的尝试实现
C++11已经支持多线程，无需调用库，用std::mutex加个锁， 把if判断放到临界区里保护起来:

```cpp
//header file
class Singleton {
public:
	static Singleton* getInstance ();
private:
	static Singleton* m_Instance;
	static mutex m_mutex;
};

//implementation.  Version 0.2  it is ok, but too expensive
Singleton* Singleton::m_Instance = nullptr;

//oops, high cost.
Singleton* Singleton::getInstance () {
	lock_guard<mutex> lock(m_mutex);
	if (m_Instance == nullptr) {
		m_Instance = new Singleton;
	}
	return Singleton;
}

```
这个版本有什么问题呢？成本太高，每个调用都去获取锁，单例创建好之后，其实已经没有必要获取锁了，并发情况下会导致其他线程因等待锁而被系统休眠，成本太高了。 那么，每次调用都加锁， 在获取锁之前再加一个if(m_Instance == nullptr)判断， 是否可行？
```cpp
//implementation.  Version 0.3
Singleton* Singleton::m_Instance = nullptr;

//oops! it does NOT work
Singleton* Singleton::getInstance () {
	if (m_Instance == nullptr) {
		lock_guard<mutex> lock(m_mutex);
		if (m_Instance == nullptr) {
			m_Instance = new Singleton;
		}
	}
	return Singleton;
}
```
想法很好，但是有严重的缺陷，来看看 m_Instance = new Singleton, 这个new操作是先分配一块空间，然后执行构造函数，相当于：

pInstance = operator new(sizeof(Singleton)); // Step 1
new (pInstance) Singleton; // Step 2

如果一个线程执行到step 1时， 另一个线程发现 m_Instance != nullptr, 直接把 m_Instance 返回，而Step 2 还没来得及执行，返回的指针指向一块并没有构造好的空间...
那么，来加一个临时变量，思路是让allocator和constructor都做完之后，再把指针赋给m_Instance，这样可行么？
```cpp
//implementation.  Version 0.4  
Singleton* Singleton::getInstance () {
	Singleton * tmp = m_Instance;
	if (m_Instance == nullptr) {
		lock_guard<mutex> lock(m_mutex);
		tmp = m_Instance;
		if (m_Instance == nullptr) {
			tmp = new Singleton;   //oops
			m_Instance = tmp;  
		}
	}
	return Singleton;
}
```
但是，我们知道，编译器优化和CPU流水线执行都有可能对代码执行顺序进行re-order.(参考[Memory Model][4])， 这样：
```cpp
//re-order之后，不能保证 step 2一定在step 3之前执行完毕。
tmp = operator new(sizeof(Singleton)); // Step 1
new (pInstance) Singleton; // Step 2
m_Instance = tmp; //Step 3
```

###3.C++11 Sequentially Consistent Atomics

要保证step 3在step 2之后执行，可以用Sequential ordering实现(即使用默认的memory_order_seq_cst)，编译器会插入memery barrier来保证。

```
#include <mutex>
#include <atomic>
using namespace std;

class Singleton {
public:
	static Singleton* getInstance ();
private:
	static std::atomic<Singleton*> m_Instance;
	static mutex m_mutex;
};

std::atomic<Singleton*> Singleton::m_Instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance () {
    Singleton* tmp = m_Instance.load ();  //memory_order_seq_cst, Sequential ordering
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);

        //maybe many threads are locked here by the mutex at the same time.
        //After the lock is free(the singleton is created by one thread), load again
        tmp = m_Instance.load(std::memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            m_Instance.store (tmp);  //memory_order_seq_cst
        }
    }
    return tmp;
}
```
那么，我们来看看， atomic的load和store是怎样保证re-order之后语义还是正确的呢？
用gcc生成汇编代码(默认是AT&T风格汇编, 我习惯看intel风格的，加个masm=intel参数):
```shell
# intel i5, ubuntu14.04, gcc 4.8.2
g++ -O2 -S -masm=intel -pthread -std=c++11  Singleton.cpp -o asm.s
```
```asm
call	_Znwm   ; call new
.LEHE0:
	mov	QWORD PTR _ZN9Singleton10m_instanceE[rip], rax  ;return value into rax
	test	rbp, rbp ; 
	mov	rbx, rax   ; rbx is the var tmp
	mfence       ;memory fence!
	je	.L11
```
可以看到编译器在x86平台上为store()生成了mfence指令。 那load()为什么没有memory fence呢，是因为x86/64是"Strong"类型的CPU(细节可参考[weak vs strong memory models][5]和Paul McKenny的文章[Memory Barriers][1]).

###4. Low-Level Ordering Constraints
一般来说，用默认的memory_order_seq_cst已经够用了，代码也简单一些。不过mfence指令的成本较高(几十倍于register to register指令，因为需要在CPU各个core和cache里进行复杂的通讯，同步cache line等等)，如果是在高并发情景下，可以考虑进一步优化。可以用low-level的acquire/release operation. 有点晦涩，可以参考[acquire and release fences][6]和[acquire and release semantics][8]

```
std::atomic<Singleton*> Singleton::m_Instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance () {
    Singleton* tmp = m_Instance.load (std::memory_order_acquire);
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock (m_mutex);
        tmp = m_Instance.load (std::memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            m_Instance.store (tmp, std::memory_order_release);
        }
    }
    return tmp;
}
```
再生成汇编代码，在x86/64平台上，可以看到，memory_order_release没有生成mfence指令。
那，为什么还要memory_order_relaxed、memory_order_release呢？ 因为它们是语言层次上的抽象，可以阻止编译器的指令re-order. 同时保证了不同CPU平台的可移植性,在ARM， PowerPC等平台会生成对应的指令。

###5. 用Template泛化

```
#include <mutex>
#include <atomic>

using namespace std;

template<typename T> class Singleton {
private:
	static atomic<T*> m_instance;
	static mutex	m_mutex;
public:
	static T* getInstance () ;
};

template<typename T> atomic<T*> Singleton<T>::m_instance;
template<typename T> mutex	Singleton<T>::m_mutex;

template<typename T> T* Singleton<T>::getInstance () {
		T* tmp = m_instance.load(std::memory_order_acquire);
		//atomic_thread_fence (std::memory_order_acquire);
		if (tmp == nullptr)	{
			lock_guard<mutex> lock(m_mutex);
			if (tmp == nullptr) {
				tmp = new T;
				//atomic_thread_fence  (std::memory_order_release);
				m_instance.store (tmp, std::memory_order_release);
			}
		}
		return tmp;
}

class Foo {};

int main()
{
	Foo* inst = Singleton<Foo>::getInstance ();
	return 0;
}
```


[1]: http://irl.cs.ucla.edu/~yingdi/web/paperreading/whymb.2010.06.07c.pdf
[2]: http://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf
[3]: http://preshing.com/20130930/double-checked-locking-is-fixed-in-cpp11
[4]: http://rsim.cs.illinois.edu/Pubs/08PLDI.pdf
[5]: http://preshing.com/20120930/weak-vs-strong-memory-models
[6]: http://preshing.com/20130922/acquire-and-release-fences
[7]: http://bartoszmilewski.com
[8]: http://preshing.com/20120913/acquire-and-release-semantics/

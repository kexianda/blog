Double-checked Locking Singleton in C++ 11.md
title: Double-checked Locking in C++ 11
tags: C++ 
category: 技术
date: 2014/12/28
---
###单线程
如何用C++实现一个Singleton呢, 不假思索，第一个版本出来
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
在单线程环境，这个版本工作的很好。但在多线程线环境下有data race了。

### 多线程的尝试实现
C++11已经支持多线程，无需调用库，用std::mutex加个锁:

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

Singleton* Singleton::getInstance () {
	lock_guard<mutex> lock(m_mutex);
	if (m_Instance == nullptr) {
		m_Instance = new Singleton;
	}
	return Singleton;
}

```
这个版本有什么问题呢？成本太高，每个调用都去获取锁，并发情况下会导致其他线程因等待锁而被系统休眠。单例创建好之后，其实已经没有必要获取锁了。那么，在获取锁之前再加一个if(m_Instance == nullptr)判读，是否可以搞定？
```cpp
//implementation.  Version 0.3
Singleton* Singleton::m_Instance = nullptr;

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
初看似乎很好，但问题很严重，来看看 m_Instance = new Singleton, 这个new操作是先分配一块空间，然后执行构造函数，相当于：

pInstance = operator new(sizeof(Singleton)); // Step 1
new (pInstance) Singleton; // Step 2

如果一个线程执行到step 1时， 另一个线程发现 m_Instance != nullptr, 直接把 m_Instance 返回，而Step 2 还没来得及执行，返回的指针指向一块并没有构造好的空间...
那么，来加一个临时变量，思路是让alloc和constructor都做完之后，再把指针赋给m_Instance，似乎很妙?
```cpp
//implementation.  Version 0.4
Singleton* Singleton::getInstance () {
	Singleton * tmp = m_Instance;
	if (m_Instance == nullptr) {
		lock_guard<mutex> lock(m_mutex);
		if (m_Instance == nullptr) {
			tmp = new Singleton;  
			m_Instance = tmp;  // 
		}
	}
	return Singleton;
}
```
初看代码，符合直觉，似乎可以工作了。但是，编译器优化和CPU执行都有可能对代码执行顺序进行re-order.(参考[Memory Model](http://rsim.cs.illinois.edu/Pubs/08PLDI.pdf)). 
```cpp
//re-order之后，不能保证 step 2一定在step 3之前执行完毕。
tmp = operator new(sizeof(Singleton)); // Step 1
new (pInstance) Singleton; // Step 2
m_Instance = tmp; //Step 3
```

### C++11 Sequentially Consistent Atomics

要保证step 3在step 2之后执行，可以用Sequential ordering实现，编译器会插入memery barrier来保证。

```
std::atomic<Singleton*> Singleton::m_Instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance () {
    Singleton* tmp = m_Instance.load ();  //memory_order_seq_cst //Sequential ordering
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_Instance.load(memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            m_Instance.store (tmp);  //memory_order_seq_cst
        }
    }
    return tmp;
}
```
```shell
g++ -O2 -S -pthread -std=c++11  masm=intel Singleton.cpp -o asm
```
```asm
call	_Znwm   ; call new 
.LEHE0:
	mov	QWORD PTR _ZN9Singleton10m_instanceE[rip], rax  ;return value rax
	test	rbp, rbp
	mov	rbx, rax
	mfence       ;memory fence!
	je	.L11
```
生成汇编代码(默认是AT&T风格汇编),可以看到编译器在x86平台上wei store插入mfence指令。danshi, sihumeiyou wei load shengcheng memory fence(dui intel strongleixingmeiyou shenru yanjiu)

### Low-Level Ordering Constraints
一般来说，用默认的memory_order_seq_cst已经够用了，代码也简单一些。不过mfence指令的成本较高，如果高并发调用频繁的话，可以考虑进一步优化。
为了取得更好的性能，可以用low-level的acquire/release operation。

```
std::atomic<Singleton*> Singleton::m_Instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance () {
    Singleton* tmp = m_Instance.load (memory_order_acquire);
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock (m_mutex);
        tmp = m_Instance.load (std::memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            m_Instance.store (tmp, memory_order_release);
        }
    }
    return tmp;
}
```
再生成汇编代码，在x86/64平台上，可以看到，shaolegemfence指令。
### 用Template泛化

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
template<typename T>mutex	Singleton<T>::m_mutex;

template<typename T> T* Singleton<T>::getInstance () {
		T* tmp = m_instance.load(memory_order_acquire);
		//atomic_thread_fence (memory_order_acquire);  //
		if (tmp == nullptr)	{
			lock_guard<mutex> lock(m_mutex);
			if (tmp == nullptr) {
				tmp = new T;
				//atomic_thread_fence  (memory_order_release);  //

				m_instance.store (tmp, memory_order_release);
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

### 参考资料
在看多线程的时候，会碰到一大堆概念，mutex, lock, atomic, memory model, memory barrier等。要更好的理解，可以先从CPU的Memory Barriers机制看起，然后看Jeff Preshing的blog, 他的帖子深入浅出，写非常好。再看看Herb Sutter,Hans Boehm等人的文章。
1. [Memory Barriers: a Hardware View for Software Hackers](http://irl.cs.ucla.edu/~yingdi/web/paperreading/whymb.2010.06.07c.pdf)
2. [C++ Memory Model - by Hans Boehm](http://rsim.cs.illinois.edu/Pubs/08PLDI.pdf)
3. [Jeff Preshing's blog](http://preshing.com/20130922/acquire-and-release-fences/)
4. [Bartosz Milewski's Programming Cafe](http://bartoszmilewski.com/)


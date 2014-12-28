Double-checked Locking in C++ 11.md
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
在单线程环境，这个版本工作的很好。多线程线显然有data race了。

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

//implementation.  Version 0.2。 it is ok, but too expensive
Singleton* Singleton::m_Instance = nullptr;

Singleton* Singleton::getInstance () {
	lock_guard<mutex> lock(m_mutex);
	if (m_Instance == nullptr) {
		m_Instance = new Singleton;
	}
	return Singleton;
}

```
这个版本有什么问题呢？成本太高，每个调用都去获取锁，并发情况下会导致其他线程因等待锁而被系统休眠。单例创建好之后，其实已经没有必要获取锁了。
在获取锁之前再加一个if(m_Instance == nullptr)岂不是大绝招？
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
这个版本有什么问题呢，来看 m_Instance = new Singleton; 这个new操作是先分配一块空间，然后执行构造函数：
pInstance = operator new(sizeof(Singleton)); // Step 1
new (pInstance) Singleton; // Step 2
如果一个线程执行到step 1时， 另一个线程发现m_Instance != nullptr, 直接把m_Instance返回，这个这个指针指向的空间并没有构造好。
那么价格临时变量， 构造好之后，再赋给m_Instance,如下：
```cpp
//implementation.  Version 0.4
Singleton* Singleton::getInstance () {
	Singleton * tmp = m_Instance;
	if (m_Instance == nullptr) {
		lock_guard<mutex> lock(m_mutex);
		if (m_Instance == nullptr) {
			tmp = new Singleton; //step 1
			m_Instance = tmp;  //step 2
		}
	}
	return Singleton;
}
```
这个版本还是有两个问题。
1. 编译器或CPU进行re-ordering后，并不能保证m_Instance = tmp时，构造函数已经执行完毕。所以需要memory barriers来保证m_Instance的赋值在构造完毕之后。保证了这个顺序，程序就工作了。
2. 要保证Singleton::m_Instance的原子性。在不同平台x86/64, PowerPC, ARM等平台下，64位指针的赋值可能不是一个汇编指令。所以要保证m_Instance的原子性。

### 用C++11的Acquire / Release Fences 
```
std::atomic<Singleton*> Singleton::m_Instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance() {
    Singleton* tmp = m_Instance.load(memory_order_relaxed); //

    atomic_thread_fence(memory_order_acquire);// acquire语意保持上面的代码re-order后不向下越过此行。 \_/
    if (tmp == nullptr) {
        lock_guard<mutex> lock(m_mutex);
        tmp = m_Instance.load(memory_order_relaxed); //relaxed 
        if (tmp == nullptr) {
            tmp = new Singleton;
            atomic_thread_fence(memory_order_release);// release语意保持下面的代码re-order后不向上越过此行。 /-\
            										//关键之处。这里保证了构造完成之后才把tmp赋给m_Instance
            m_Instance.store(tmp, memory_order_relaxed);
        }
    }
    return tmp;
}
```

### C++11 Sequentially Consistent Atomics
```
std::atomic<Singleton*> Singleton::m_Instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance () {
    Singleton* tmp = m_Instance.load ();  //memory_order_seq_cst
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_Instance.load(memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            m_Instance.store (tmp);
        }
    }
    return tmp;
}
```

### 用C++11 Low-Level Ordering Constraints
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
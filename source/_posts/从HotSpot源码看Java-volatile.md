---
title: 从HotSpot源码看Java volatile
date: 2017-04-28 22:51:49
tags:
 - HotSpot
 - 内存模型
category: 技术
---

近期有人和我讨论java volatile, 人老脑子懵，我还volatile不是atomic的云云，神经还搭在C++ 的volatile坑里没恢复过来, 过来一阵才反应过来. 就此契机,整理一下. 
几年前曾给同事做过一个java内存模型的knowledge sharing, 摘取部分内容放这, 简单的回顾下内存模型, 然后从JVM hotSpot实现的角度来仔细看看volatile的语义.
<!-- more -->

### 1. reorder和memory barrier
指令执行的reorder有两个原因，一是编译器的优化，二是CPU执行的优化.
cpu硬件优化的out-of-order机制会导致指令的reorder. 另外，CPU的各个核有自己的registers和cache(L1,L2,L3)，cache更新一致性协议会导致reorder，从cache角度更明了的体现了memory model中visibility概念的来源.

为了避免reorder，保证逻辑正确性，我们需要memory barrier.
有读(Load)有写(Store), 组合成四种基本的reorder类型(memory barrier类型)
* LoadLoad 
* LoadStore
* StoreLoad 
* StoreStore

还会看到acquire/release，为啥弄出这两概念呢，不妨从应用场景来理解.
1. Critical Section，
```
EnterCriticalSection
   acquire  semantics
 /--------------------------------------------\
/       do critical job                        \ 
  all memory operations stay below the line

  all memory operations stay above the line
\                                                 /
 \-----------------------------------------------/
LeaveCriticalSection

```

2. 两个线程之间的同步
```
thread1:
    result = 100
    flag = true
 \                 /
  \---------------/ (release语义)

                     thread2:                   
                    /--------------------\  (acquire语义)
                   / if (flag)            \  
                     print (result)
```
可以看到,
* acquire == LoadLoad  | LoadStore 
* release == StoreStore| LoadStore
这样，acquire/release概念就不晦涩了. btw, C++ 11里支持low-level的acquire/release语义.

### 2. x86/64 CPU的Memory Model
从Intel手册里能看到:
* Reads are not reordered with other reads. 
不需要特殊fence指令就能保证LoadLoad
* 2.Writes are not reordered with older reads.  
不需要特殊fence指令就能保证LoadStore
* 3.Writes to memory are not reordered with other writes.
不需要特殊fence指令就能保证StoreStore
* 4.Reads may be reordered with older writes to different locations but not with older writes to the same location.
需要特殊fence指令才能保证StoreLoad, 有名的Peterson algorithm算法就是需要StoreLoad的典型场景

看下HotSpot中memory barrier的实现，除了storeload， 其他的barrier在x86上并不需要cpu的barrier指令，这里是c++代码，只是一个compiler_barrier告诉gcc别瞎优化了.
```C++
// (java 9) hotspot/src/os_cpu/linux_x86/vm/orderAccess_linux_x86.inline.hpp
inline void OrderAccess::loadload()   { compiler_barrier(); }
inline void OrderAccess::storestore() { compiler_barrier(); }
inline void OrderAccess::loadstore()  { compiler_barrier(); }
inline void OrderAccess::storeload()  { fence();            }

inline void OrderAccess::acquire()    { compiler_barrier(); }
inline void OrderAccess::release()    { compiler_barrier(); }

inline void OrderAccess::fence() {
  if (os::is_MP()) {
    // always use locked addl since mfence is sometimes expensive
#ifdef AMD64
    __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
#else
    __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
#endif
  }
  compiler_barrier();
}
```

### 3. Java volatile
规范里说:
1. reads & writes act as aquire & release
2. make non atomic 64-bit operations atomic: long and double
3. writing that variable become visible to another thread 
有了reorder和memory barrier的概念和对x86 CPU的了解后，来看看在HotSpot在x86平台上volatile的实现，不妨只看对volatile double类型写的处理部分

```C++
// hotspot/src/cpu/x86/vm/templateTable_x86.cpp

void TemplateTable::putfield_or_static() {
 
   // field addresses
   const Address field(obj, off, Address::times_1, 0*wordSize);
   
   // 省略其他代码...

  // [jk] not needed currently on X86_64
  // volatile_barrier(Assembler::Membar_mask_bits(Assembler::LoadStore |
  //                                              Assembler::StoreStore));

   // Replace with real volatile test
    __ push(rdx);
    __ push(rax);                 // Must update atomically with FIST

    // [Xianda]: 把要赋的值从栈顶load到浮点运算单元寄存器
    __ fild_d(Address(rsp,0));    // So load into FPU register

    // [Xianda]: 把值从到浮点运算单元寄存器写到field的地址. 指令保证原子性
    // [Xianda]: fistp_d on x86, movsd on x64
    __ fistp_d(field);            // and put into memory atomically
    
    __ addptr(rsp, 2*wordSize);

    // [Xianda]: 加入memory-barrier保证语义，
    // volatile变量的写，后面的指令不能被reorder这个store之前，
    // 保证这个写对其他操作是可见的
    // volatile_barrier();
    volatile_barrier(Assembler::Membar_mask_bits(Assembler::StoreLoad |
                                                 Assembler::StoreStore));
}
```

从代码和注释能看到，原子性的保证，还有内存屏障的语义的实现.
写完后加的StoreLoad | StoreStore(注意这里是汇编的barrier了，没compiler的事)，第2小节里我们知道StoreLoad需要一个lock，这个指令x86上可以做为memory barrier指令.

### 4. Java volatile 实验 (show me the code...)
```java
//debug hotspot: break TemplateTable::volatile_barrier
class VolatileTest {
  private static volatile long volatileVar;

  public static void main(String[] args) {
    test();
  }
  public static void test(){
    long tmp = volatileVar + 5; //volatile load

    volatileVar = tmp * tmp;  //volatile store;
    // fistp_d on x86, movsd on x64
  }
}
```

```shell
// 需要下载工具hsdis-amd64.so, 放到${LD_LIBRARY_PATH}里
// -XX:CompileCommand=dontinline,*VolatileTest.test的意思是,不要对test inline化, 通配符*通配package名
// -XX:CompileCommand=compileonly,*VolatileTest.test 
// java 9里ok, 7,8可能不行
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -Xcomp -XX:CompileCommand=dontinline,*VolatileTest.test -XX:CompileCommand=compileonly,*VolatileTest.test VolatileTest

```

把java运行时的cpu指令打印出来： 
```asm
vmovapd %xmm0,%xmm1     ; 加
vmulsd %xmm1,%xmm1      ; 乘
vmovsd %xmm1,0x68*(%rsi) ;  用xmm寄存器写回内存, 原子性
lock addl $0x0, 0x40(rsp) ; putstatic volatileVar
                          ; VolatileTest::test()

```

### 5. C++ volatile != Java volatile
Java volatile其实和C++ std::atomic<T>有些类似. 详见Herb Sutter[文章](http://www.drdobbs.com/parallel/volatile-vs-volatile/212701484)

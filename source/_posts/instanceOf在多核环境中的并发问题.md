---
title: instanceOf在多核环境中的并发问题
date: 2017-06-09 22:23:56
tags:
 - HotSpot
 - 并发
---
同事分享了通过profiling发现的高并发性能问题, [一个Spark的性能问题](https://www.slideshare.net/SparkSummit/accelerating-spark-genome-sequencing-in-clouda-data-driven-approach-case-studies-and-beyond-spark-summit-east-talk-by-lucy-lu-and-eric-kaczmarek) 和[一个Cassandra的性能问题](https://issues.apache.org/jira/browse/CASSANDRA-12787), 涉及到:
- Scala Scalability Issue
- instanceOf在HotSpot中的实现和优化
- CPU Cache thrashing / NUMA  

从应用层(大数据)到JVM(HotSpot)instanceOf的实现, 再到CPU的硬件机制, 涉及到知识很有意思, 对写高并发高性能代码有启发, 我把涉及到的知识拓展和小结一下.
<!--more-->

## 1. CPU Cache & Thrashing & NUMA
#### Intel x86 CPU Cache
首先看看现代的x86 CPU访问不同layer数据的时间, 先有个感性认识.  

| 存储空间         | 时间成本   |
| :--------------: | :----------: |
| Registers/buffers  |  ~ 1 cycle < 1ns  |
| L1 cache  |  ~ 3 cycles  ~1ns  |
| L2 cache  |  ~ 12 cycles  ~3ns  |
| L3 cache  |  ~ 38 cycles < 12ns  |
| QPI*  |  ~40 ns  |
| DRAM  |  ~65 ns  |

QPI(Intel QuickPath Interconnect)是用来链接不同处理器. 这里有篇文章[A Deeper Look Inside Intel QuickPath Interconnect](http://www.drdobbs.com/go-parallel/article/print?articleId=222301437)), 处理器间cache coherence通过QPI来处理. 随着core/socket(processor)的增加, cache coherence更加复杂和成本也在上升.

#### Thrashing
访问memory和cache之间存在很大的差距, 如果没有cache, CPU快也没有用只能干等. 一般而言, 现在的x86 CPU有非常好cache hit rate(90%+), 可参考[这篇文章](https://www.extremetech.com/extreme/188776-how-l1-and-l2-cpu-caches-work-and-why-theyre-an-essential-part-of-modern-chips).  
特殊情况下, 数据被load到cache, 又迅速的被丢弃, 会导致效率低下([CPU cache thrashing](https://pomozok.wordpress.com/2011/11/29/cpu-cache-thrashing/)). 非常类似操作系统中内存管理中的Thrashing概念. cahce line不停换进换出.
一个例子演示下:
```c
for (int i=0; i<SIZE; i++) {
    for (int j = 0; i < SIZE; j++) {
        //array[i][j] = i * j;  //
        array[j][i] = i * j;  // cache thrashing.
    }
}
```

#### NUMA
关于NUMA, HP linux kernel team有个slides([Optimizing Application Performance in Large Multi-core Systems](http://events.linuxfoundation.org/sites/events/files/slides/Optimizing%20Application%20Performance%20in%20Large%20Multi-core%20Systems_0.pdf)).

另, [这篇](http://cenalulu.github.io/linux/numa/)也可以参考.

## 2. instanceOf 在HotSpot中的优化和问题
### instanceOf的语义和实现
[RednaxelaFX](https://www.zhihu.com/people/rednaxelafx/)在知乎有个很好的[回答](https://www.zhihu.com/question/21574535), R大由浅入深, 然后深入得只能JVM开发者才能懂:-)

>作者：RednaxelaFX
链接：https://www.zhihu.com/question/21574535/answer/18998914
来源：知乎
著作权归作者所有。

>简单来说，优化的主要思路就是把Java语言的特点考虑进来：由于Java的类所继承的超类与所实现的接口都不会在运行时改变，整个继承结构是稳定的，某个类型C在继承结构里的“深度”是固定不变的。也就是说从某个类出发遍历它的super链，总是会遍历到不变的内容。这样我们就可以**把原本要循环遍历super链才可以找到的信息缓存在数组里，并且以特定的下标从这个数组找到我们要的信息**。同时，Java的类继承深度通常不会很深，所以为这个缓存数组选定一个固定的长度就足以优化大部分需要做子类型判断的情况。**HotSpot VM具体使用了长度为8的缓存数组，记录某个类从继承深度0到7的超类。HotSpot把类继承深度在7以内的超类叫做“主要超类型”（primary super），把所有其它超类型（接口、数组相关以及超过深度7的超类）叫做“次要超类型”（secondary super）。对“主要超类型”的子类型判断不需要像Kaffe或JamVM那样沿着super链做遍历，而是直接就能判断子类型关系是否成立。这样，类的继承深度对HotSpot VM做子类型判断的性能影响就变得很小了。对“次要超类型”，则是让每个类型把自己的“次要超类型”混在一起记录在一个数组里，要检查的时候就线性遍历这个数组。**留意到这里把接口类型、数组类型之类的子类型关系都直接记录在同一个数组里了

展示一段代码, 逻辑基本就是R大描述的过程.
```c++
// hotspot/src/share/vm/oops/klass.cpp
class Klass : public Metadata {

    //...

    // Cache of last observed secondary supertype
    Klass*      _secondary_super_cache;
    // Array of all secondary supertypes
    Array<Klass*>* _secondary_supers;
    // Ordered list of all primary supertypes
    Klass*      _primary_supers[_primary_super_limit];


    bool is_subtype_of(Klass* k) const {
       juint    off = k->super_check_offset();
       Klass* sup = *(Klass**)( (address)this + off );
       const juint secondary_offset = in_bytes(secondary_super_cache_offset());
       if (sup == k) {
         return true;
       } else if (off != secondary_offset) {
         return false;
       } else {
         return search_secondary_supers(k);
       }
     }
}

bool Klass::search_secondary_supers(Klass* k) const {
  // Put some extra logic here out-of-line, before the search proper.
  // This cuts down the size of the inline method.

  // This is necessary, since I am never in my own secondary_super list.
  if (this == k)
    return true;
  // Scan the array-of-objects for a match
  int cnt = secondary_supers()->length();
  for (int i = 0; i < cnt; i++) {
    if (secondary_supers()->at(i) == k) {
      ((Klass*)this)->set_secondary_super_cache(k); // 注意这里, 设置_secondary_super_cache
      return true;
    }
  }
  return false;
}
```

### instanceOf的并发性能问题
这里关心的点, 就是HotSpot源码中这个_secondary_super_cache被设置. instanceOf调用时, _secondary_super_cache不停的被设置, 在多核或多CPU高并发环境里.
比如下面情况, 有个基础的类class A, instanceOf操作极频繁:
```
A instanceOf super1;  // running on processor 1, core 1
A instanceOf super2;  // running on processor 1, core 2
A instanceOf super3;  // running on processor 2, core 1
A instanceOf super4;  // running on processor 2, core 2
```
super1被设置到_secondary_super_cache, CPU core把这个值换进到cache. 然而很快被换出了, 因为另外一个core设置新的值super2了. 而各个core/processor需要保持cache coherence, super1的cache line被换出. core/processor个数越多, 并发度越高, cahce line不停换进换出, 相互拖累程度越严重. (有了前三节的介绍, 这个理解起来不难了.)


#### Scala lang  
Scala集合类的实现里, scala.collection.mutable.Builder.sizeHint()里有类型判断.
```scala
def sizeHint(coll: TraversableLike[_, _]) {
    if (coll.isInstanceOf[collection.IndexedSeqLike[_,_]]) {
      sizeHint(coll.size)
}
```
这导致了并发性能问题, Scala在最新的2.12.x版本里改了实现, 细节见[JIRA](https://issues.scala-lang.org/browse/SI-9823). 在测试环境, 优化后, 有3X的性能差距.

#### Cassandra
Cassandra也有个类似的问题, 见[Reduce instanceOf() type checking to improve performance](https://issues.apache.org/jira/browse/CASSANDRA-12787)

这两个问题, 分别是在scala/java代码层面修了bug, 或许JVM将来会进一步改进改善这个问题.  

/// 完

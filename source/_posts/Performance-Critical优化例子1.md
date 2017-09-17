---
title: Performance-Critical的优化例子(1)
tags:
  - 汇编优化
categories: 技术
date: 2017-09-10 00:00:00
---

搬砖时, 很少碰到需要写发挥CPU极致性能的代码. 一般而言, 软件系统的性能瓶颈在上层的架构或算法,IO,内存带宽上. 对CPU优化的工作都一般都是编译器,虚拟机和底层各种runtime来处理.
当瓶颈在CPU上时, 比如特定领域的加密/压缩/视频图形解码等, 汇编优化很重要. 在大数据/云时代那些大规模部署的分布式程序能有个百分之几的优化省下不少的服务器, 一些关键的hotspot函数还是值得优化一下. 有时候得汇编加持, 追求极致性能.

一些汇编的奇技淫巧, 对日常搬砖没啥作用. 但因此而去了解一下CPU的front-end对我们用高级语言(C/C++/Java)搬高性能代码有些好处.  
<!--more-->

闲扯下, 比如Spark/Impala等的code-gen, 就有Cache-aware的优化. 指令cache L1在现代x86CPU上是32K. 一个简单的表达式evaluation如果调用多次虚函数, 32K指令cache L1命中率就低了. 生成的代码减少虚函数调用, 除了减少访问内存次数, 有更好Locality, 指令cache L1命中率提升很多, 能带来提升巨大的性能. Spark的Tungsten项目之前, CPU居然也是性能瓶颈,计算浪费太大.

盗两张图, 看看CPU Front-End/Back-End:
x86 Microarchitecture:
![Microarchitecture](https://software.intel.com/sites/default/files/did_feeds_images/6E3B86B7-8072-4E6A-9E27-0CD21446ACB1/6E3B86B7-8072-4E6A-9E27-0CD21446ACB1-imageId=92436631-C87A-402A-9EBD-12E8BA82D8B2.png)
Xeon Skylake:
![Xeon Skylake](https://img.purch.com/500-png/w/755/aHR0cDovL21lZGlhLmJlc3RvZm1pY3JvLmNvbS9SL1MvNjkxNzY4L29yaWdpbmFsLzUwMC5QTkc=)

粗糙的列下CPU的front-end和features的一些有用的点:
* 各个cache level的大小和访问时间,
* out-of-order window
* ILP, 指令间的并行,往往能带来几倍的性能提升.
* 分支预测
* 取码/解码
* 数据/指令的对齐(alignment)
* SIMD等features: SSE/AVX/AVX2, 以及在牙膏厂最新CPU(Skylake)的AVX512

最近一个项目需要在最新的Xeon Skylake上优化memcpy, 小结下需要那些奇技淫巧:
x86上,不同指令的时钟周期成本可能不同(x86), 甚至同一个指令,在不同mirco-architecture下的成本也可能不一样, 需要查牙膏厂的software developer's manuals. 其实CPU底层的实现非常非常的复杂, 硬件对于码农是一个黑盒, 手册上往往也是寥寥数语, 这种就十分头疼了.

* 跳转表, 对不同长度对于不同代码,直接跳转到特定逻辑, 减少长度判断/跳转(jmp)次数, 减少branch-prediction失败次数. 编译器在优化switch-case时也是这么干的.
* 不同长度采取不同算法, 比如rep movsb在新的CPU上性能不错, 适合一定长度的内存拷贝, 但是rep movsb指令在CPU硬件层准备成本较高, 太短的内存拷贝就不划算了. 这个特性过于的依赖特定平台了.
* 预取数据到cache. prefetcht0/prefetcht1/prefetcht2/prefetchnta指令. CPU的Cache是CPU硬件管理的,这几个指令对CPU而言是个hint. 如何更好的使用,提前多久预取数据, intel手册对此没有详细解释.
* 较大数据拷贝时, 写时不污染CPU Cache. 往内存写数据时,一般先读入cpu cache, 写入cache,再写入内存. 如果数据不需要cache起来, 可以直接写入内存, CPU会把写入的数据合并成块写入. 所以这时候x86的strong内存模型就不"strong"了, 需要内存屏障store fence(sfence指令).
* AVX-512的 zmm16~zmm31寄存器. 为了避免AVX-512到AVX/SSE转换成本, 不用zmm1~zmm15寄存器.

(待续...)

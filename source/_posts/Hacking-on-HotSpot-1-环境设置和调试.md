---
title: Hacking on HotSpot (1) 环境设置和调试
date: 2017-06-03
tags:
 - HotSpot
category: 技术
---
调试HotSpot的环境配置的资料确实比较少. 想提交patch或者有时候好奇需要深入一下JVM看看具体的实现, 需要配置下开发环境. 这里记录整理一下.
有人写过Mac上配置, Windows的环境没查到,我也没弄过, 估计宇宙第一IDE的Visual Studio应该很好使:-)
<!--more-->

## 0. prerequisites
#### mercurial
OS环境我选Ubuntu 1604. 首先安装OpenJDK用的SCM工具mercurial
```shell
sudo apt-get install mercurial

```
再安装个mercurial pluggin
```
mkdir ~/jdk/tools && cd ~/jdk/tools  # 目录随意...
wget http://hg.openjdk.java.net/code-tools/trees/raw-file/tip/trees.py
```
vim ~/.hgrc
```
[extentions]
Trees= ~/jdk/tools/trees.py
```
#### clone repo
```
hg tclone http://hg.openjdk.java.net/jdk9/dev jdk9-dev
```

#### hsdis-amd64.so
这个工具用来输出运行时的汇编, 网上下载hsdis-amd64.so, 把路径添加到LD_LIBRARY_PATH.


## 2. 编译OpenJDK 9
在centos6上碰到不少问题, 在Ubuntu1604比较顺利, 我在多台Ubutun上都很顺利,没有碰到奇怪的问题. 相比以前的jdk7, 现在jdk9编译起来顺利多了.  先安装需要的依赖:
```
sudo apt-get install libx11-dev libxext-dev libxrender-dev libxtst-dev libxt-dev
sudo apt-get install libcups2-dev libfreetype6-dev libasound2-dev
sudo apt-get install libelf-dev
```

```
cd ~/jdk/jdk9-dev
bash configure --with-native-debug-symbols=internal --with-debug-level=slowdebug

# 或者建另一个目录
# mkdir ../build && cd ../build && bash ../jdk9-dev/configure --with-native-debug-symbols=internal --with-debug-level=slowdebug
```
--with-debug-level=slowdebug 这个选项是编译debug版本.

如果还缺依赖, configure时会报出来,按提示安装即可. make clean 一下再configure.  
注意, 这些--with-xxx-xxx的参数, 可能会变, 如果报错, 去autoconf目录里grep一把, 具体解决.



## 调试

#### netbeans
用netbeans打开 common/nb_native/下的netbeans工程就可以了. 右键在Projects区里的工程, 设置Debug参数, 比如
```
Debug command:  ${OUTPUT_PATH} XxJavaApp
```
#### gdb
比如用gdb脚本:
cat myscript.gdb
```
file ./build/linux-x86_64-normal-server-slowdebug/images/jdk/bin/java

# your args...
set args "-Xbatch -XX:+UnlockDiagnosticVMOptions -Xcomp "

set breakpoint pending on
break InterpreterRuntime::monitorenter

run MyJavaApp
```

```
gdb --command=myscript.gdb
```
## 3. 例子
运行上面的gdb脚本, 断点处停下来, 看看call stack,
```
(gdb) bt
#0  InterpreterRuntime::monitorenter (thread=0x7ffff0019000, elem=0x7ffff7fd4548) at /home/xianda/jdk/jdk9-dev/hotspot/src/share/vm/interpreter/interpreterRuntime.cpp:635
#1  0x00007fffd788cc19 in ?? ()
#2  0x00007fffd788caca in ?? ()
#3  0xffffffff98030002 in ?? ()
#4  0x00000006d6d47bf0 in ?? ()
#5  0x00007ffff7fd4548 in ?? ()
#6  0x00007fffa199f986 in ?? ()
#7  0x00007ffff7fd45c0 in ?? ()
#8  0x00007fffa19a17e0 in ?? ()
#9  0x0000000000000000 in ?? ()
```
咦, 这里的"??"是什么, 我只是想看看call stack...
做HotSpot的人早帮我们想好了, 在gdb里执行:
```
(gdb) call pns($rsp, $rbp, $pc)
```

```
"Executing pns"
Native frames: (J=compiled Java code, A=aot compiled Java code, j=interpreted, Vv=VM code, C=native code)
V  [libjvm.so+0xc573c6]  InterpreterRuntime::monitorenter(JavaThread*, BasicObjectLock*)+0x1a
j  jdk.internal.misc.VM.initLevel(I)V+6 java.base
j  java.lang.System.initPhase1()V+138 java.base
v  ~StubRoutines::call_stub
V  [libjvm.so+0xc71794]  JavaCalls::call_helper(JavaValue*, methodHandle const&, JavaCallArguments*, Thread*)+0x6a2
V  [libjvm.so+0x104af5f]  os::os_exception_wrapper(void (*)(JavaValue*, methodHandle const&, JavaCallArguments*, Thread*), JavaValue*, methodHandle const&, JavaCallArguments*, Thread*)+0x41
V  [libjvm.so+0xc710da]  JavaCalls::call(JavaValue*, methodHandle const&, JavaCallArguments*, Thread*)+0xaa
V  [libjvm.so+0xc70bcf]  JavaCalls::call_static(JavaValue*, KlassHandle, Symbol*, Symbol*, JavaCallArguments*, Thread*)+0x169
V  [libjvm.so+0xc70cff]  JavaCalls::call_static(JavaValue*, KlassHandle, Symbol*, Symbol*, Thread*)+0x9d
V  [libjvm.so+0x122d226]  call_initPhase1(Thread*)+0xb6
V  [libjvm.so+0x122d7dd]  Threads::initialize_java_lang_classes(JavaThread*, Thread*)+0x2bd
V  [libjvm.so+0x122e16b]  Threads::create_vm(JavaVMInitArgs*, bool*)+0x695
V  [libjvm.so+0xcbdfca]  JNI_CreateJavaVM_inner(JavaVM_**, void**, void*)+0xec
V  [libjvm.so+0xcbe360]  JNI_CreateJavaVM+0x41
C  [libjli.so+0x6eb4]  InitializeJVM+0x13a
C  [libjli.so+0x3d0e]  JavaMain+0xc6
C  [libpthread.so.0+0x76ba]  start_thread+0xca
```
包括Java代码的call stack都清楚的出来了.

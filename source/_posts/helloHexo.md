title: hello, hexo
tags: hexo
category: 随笔
date: 2014/11/10

---
### **选择hexo**

虽然是一个蓬头垢面的码农，从审美角度，接受不了公共blog的UI和页面边上的无关广告:-)
<a href="https://github.com/hexojs" target="_blank">hexo</a>不错，是个静态blog工具，可以支持markdown格式。 还有个极简的light theme，比较喜欢，稍做定制. 改了部分css， 把JQuery的google CDN改了. 决定把blog迁移到过来了.

副标题盗取自荷尔德林的诗歌《在柔媚的湛蓝中》.

<!-- more -->


### **测试代码**
测试一把代码高亮功能。

C++:
```cpp
#if defined(_WIN32) || defined(_WIN64)
#define _CRTDBG_MAP_ALLOC
#include <stdlib.h>
#include <crtdbg.h>
#endif //Windows CRT GBD

#ifdef __linux__
#include <cstdlib>
#include <mcheck.h>
#endif //__linux__

#include <iostream>
using namespace std;

int main () {

	#ifdef __linux__
	//$mtrace test.out memcheck.o > report.o
	setenv ("MALLOC_TRACE", "memcheck.o", 1);
	mtrace();
	#endif //__linux__

	cout<<"hello"<<endl;

	return 0;
}
```

JavaScript:
```javascript
(function(){
    console.log("hello, world!");
})()
```
Java:
```java
package kexianda.test;
public class Test {
    public static void main(String[] args){
        System.out.println("hello");
    }
}
```

Python:
```python
print "hello, world"
```

收工。

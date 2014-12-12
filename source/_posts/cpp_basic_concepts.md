title: C++ basic concepts (1)
tags: C++
category: 技术
date: 2014/11/20
---

这篇先简单讨论C++中的几个basic concepts: lvalue, rvalue，lvalue reference, rvalue reference，cv-qualified, reference collapsing rule等等。这是理解template argument deduction的基础。

<!-- more -->


## 1. Expression category taxonomy (lvalue, xvalue, pvalue)

C++98,03里有lvalue和rvlaue的概念。

简单的说，lvalue一个在内存有确定地址的对象，可以寻址。而rvalue是一个临时对象。

>An lvalue (locator value) represents an object that occupies some identifiable location in memory (i.e. has an address).

--cited from [here][2]

看例子：

``` cpp
int a = 1;  //a is an lvalue,  1 is an constant

// 1 = a; //Error!  1 is rvalue.

//(var + 1) = 0; //Error.   expression (var + 1) is an rvalue.
```

Ok, 到此，很熟悉很简单。 但是,不复杂不舒服斯基的C++在C++11里进行了更细致的分类，增加了三个categories
* rvalue
* lvalue
* xvalue
* glvalue
* prvalue

为什么这么搞？
因为C++11引进了rvalue reference，在overload resolution，template argument deduction等地方，需要对expression的进行更细的分类。
它们之间的关系，看Venn图。from [这里][3]

```
----- Expression category taxonomy  -----
   ______ ______
   /      X      \
  /      / \      \
 |   l  | x |  pr  |
  \      \ /      /
   \______X______/
       gl    r

 // l is lvalue, pr is prvalue, x is xvalue, gl is glvalue, r is rvalue.
```
上图lvalue和rvalueC++11大致和C++98是一样的，只不过把rvalue分为了xvalue和pvalue. 而glvalue分为xvalue和lvalue
再看标准，
>Every expression belongs to exactly one of the fundamental classifications in this taxonomy: lvalue,
xvalue, or prvalue. This property of an expression is called its value category.  (N3690)

一个expression的value category有三种: lvalue， prvalue, xvalue，这三种为基本的value category。
接下来看严谨一点儿的定义，但不完全，先忽略不常用的情况。

### lvalue
an lvalue is an expression that identifies a non-temporary object or a non-member function.

>([N3690 3.10][1])
An lvalue (so called, historically, because lvalues could appear on the left-hand side of an assignment expression) designates a function or an object. [ Example: If E is an expression of pointer type, then *E is an lvalue expression referring to the object or function to which E points. As another example, **the result of calling a function whose return type is an lvalue reference is an lvalue**. —end example ]

简单起见，只列出常见的情况：
1. The name of a variable or function in scope, regardless of type, even if the variable's type is rvalue reference, the expression consisting of its name is an lvalue expression.
2. Function call expression if the function's return type is an lvalue reference
3. Built-in pre-increment and pre-decrement, dereference, assignment and compound assignment, subscript (except on an array xvalue)
4. Cast expression to lvalue reference type. static_cast<int&> x
5. String literal

```cpp
int a;  //i is an lvalue
int&& arref = a;  //arref is an lvalue, but it's type is 'rvalue reference'
++a;   //++a is an lvalue

int arr[] = {1,2};

int* p = &arr[0];  //arr, arr[0] and p are both lvalue

*(p+1) = 3;  //dereference.   expression (p+1) is an rvalue,  but *(p+1) is an lvalue

"hello";     //"hello" is an lvalue
```

### pvalue

A prvalue ("pure" rvalue) is an expression that identifies a temporary object (or a subobject thereof) or is a value not associated with any object.

下列expressions是prvalues:
1. Literal (except string literal), such as 42 or true or nullptr.
2. Function call  if the function's or the overloaded operator's return type is not a reference,
such as str.substr(1, 2) or str1 + str2
3. Built-in post-increment and post-decrement, arithmetic and logical operators, comparison operators, address-of operator
4. Cast expression to any type other than reference type.
5. Lambda expressions, such as [](int x){return x*x;}  (since C++11)


```cpp
23L // 23L is an pvalue

int a;
a++;  //a++ is an pvalue

(1+2); // (1+2) is an pvalue

(int) 1.2;  // (int)1.2

```


### xvalue

>An xvalue (an “eXpiring” value) also refers to an object, usually near the end of its lifetime (so that its
resources may be moved, for example). An xvalue is the result of certain kinds of expressions involving
rvalue references (8.3.2). [ Example: The result of calling a function whose return type is an rvalue
reference is an xvalue. —end example ]


>An xvalue is an expression that identifies an "eXpiring" object, that is, the object that may be moved from. The object identified by an xvalue expression may be a nameless temporary, it may be a named object in scope, or any other kind of object, but if used as a function argument, xvalue will always bind to the rvalue reference overload if available.

下列expressions是xvalues:
1. A function call if the function's or the return type is an rvalue reference to object type, such as std::move(val)
2. A cast expression to an rvalue reference to object type, such as static_cast<T&&>(val) or(T&&)val

Like prvalues, xvalues bind to rvalue references, but unlike prvalues, an xvalue may be polymorphic. and a non-class xvalue may be cv-qualified. (since C++11)

glvalue和rvalue就不谈，可看ISO的标准文档。到现在，看下面例子，应该很好理解了。

```cpp
int   prvalue();

int&  lvalue();

int&& xvalue();
```

有一点需要注意，**“class rvalues can have cv-qualified types, but built-in types (like int) can't ”** 在研究tempalte argument deduction的时候，这一点曾让我迷惑过，虽然现在看起来比较自然，看代码：

``` cpp
struct Foo {};

Foo f() { return Foo(); }
const Foo cf() { return Foo(); }

int b() {int i=0; return i;}; //return built-in type rvalue

const int cb() {int i=0; return i;} // try to add const

template<typename T> void g(T&& t) {}

int main ()
{
        g (f());   //calls: g<Foo>(Foo&&)
        g (cf());  //calls: g<Foo const>(Foo const&&).
        g (b());   //calls: g<int>(int&&)
        g (cb());  //calls: g<int>(int&&) Notice:no const version!
        return 0;
}

```

若不理解T是咋deduce出来的，可暂且不管. 通过对比，至少可以得出built-in type rvalue没有cv-qualifiyed版本。


## 2. C++ Types

C++的Types分为三种：**Fundamental types**, **Compound Types**, **CV-qualifiers**
**Fundamental types:**是指bool, int, char 等等。

**Compound Types:**包括：
* arrays, pointers, functions
* references (lvalue reference 和 rvalue reference)
* classes, unions

**CV-qualifiers:** 指 const qualifier和volatile qualifier

## 3. lvalue reference / rvalue reference

有了上面Types的分类，再来研究下lvalue reference和rvalue reference。
[C++ ISO文档][1]里谈到的术语reference，包含了两种，lvalue reference和rvalue reference.

lvalue reference语法形式是 T&， rvalue reference是 T&&。 rvalue reference和lvalue reference差不多，特殊地方在于rvalue reference能绑定到一个临时变量上，而非const的rvalue reference不能。

``` cpp
A a;
A& a_ref = a;  // an lvalue reference

A a;
A&& a_rref = a; // a_rref's type of rvalue reference
              // but a_rref is an lvalue, because the named rvalue references are lvalues.

void foo(int&& t) {
  // t is initialized with an rvalue expression
  // but is actually an lvalue expression itself, here t is a named rvalue reference.
}

A&  a_ref3 = A();  // Error!
A&& a_ref4 = A();  // Ok

```

a_rref's type of rvalue reference but a_rref is an lvalue, because the named rvalue references are lvalues. 稍有晦涩。术语lvalue / rvalue 是表达式的分类(value categories of Expressions), 不是C++的Types(类型);而lvalue reference/rvalue reference是compound type是一种type.


## 4. reference collapse rule

>If a typedef (7.1.3), a type template-parameter (14.3.1), or a decltype-specifier (7.1.6.2) denotes a type TR

that is a reference to a type T, an attempt to create the type “lvalue reference to cv TR” creates the type

“lvalue reference to T”, while an attempt to create the type “rvalue reference to cv TR” creates the type TR.

-- cited from [ISO N3690][1]

``` cpp
typedef int&  lref;
typedef int&& rref;
int n;
lref&  r1 = n; // type of r1 is int&
lref&& r2 = n; // type of r2 is int&
rref&  r3 = n; // type of r3 is int&
rref&& r4 = 1; // type of r4 is int&&

```
这个规则和template argument deduction 中的T&& special case, 就是std::forward实现的基础。



## 5. Recommended References
0. [value category](http://en.cppreference.com/w/cpp/language/value_category)

1. [Universal References in C++11](http://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers)  -- by Scott Meyers




[1]: http://isocpp.org/files/papers/N3690.pdf

[2]: http://eli.thegreenplace.net/2011/12/15/understanding-lvalues-and-rvalues-in-c-and-c

[3]: http://stackoverflow.com/questions/3601602/what-are-rvalues-lvalues-xvalues-glvalues-and-prvalues






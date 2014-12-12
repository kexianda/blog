title: template argument deduction
tags: C++ 
category: 技术
date: 2014/11/25
---

C++11加入了rvalue reference的概念后，类型推导规则更加复杂了。微软的Stephan Lavavej在Channel9的Core C++系列中对deduction简单地做了些[介绍](http://channel9.msdn.com/Series/C9-Lectures-Stephan-T-Lavavej-Core-C-/Stephan-T-Lavavej-Core-C-2-of-n) ，但并没有深入地讲。我拿[C++标准][1]研究了一把，发现deduction的规则细节还是比较繁琐的。标准文档的可读性不太好，颇费脑细胞. 相对而言[cppreference][2]可读性要好一些。这篇主要总结函数调用时的类型推导(Deduction from a function call)和不进行类型推导的几种情况(Non-deduced contexts)

<!-- more -->

如果不了解lvalue, xvalue和pvalue. 可参考[前一篇][3].

## Deduction from a function call
先说明一下几个术语。模板函数的形数parameter简称为 P，实参argument简称为 A. 编译器推导是指根据P/A pair推导出P的真正Type。 推导过程中，通过一定规则对P/A进行调整. P调整后称为deduced A, A调整后称为tranformed A.  

推导规则
一般来说deduced A==tranformed A，如果不相同，再由额外规则处理。多个参数的情况，则对每个P/A pair分别进行推导，如果有不一致，则失败。有一点，在标准里没找到（我没有通读），if A is reference type，*tranformed A* is the type referred by A. 

### (Rule 1) if P is NOT a reference type
>(rule 1): if P is NOT a reference type: ([N3690][1], 14.8.2.1/2)
* 1.1 A is array, array-to-pointer conversion
* 1.2 A is function type, function-to-pointer conversion
* 1.3 if A is cv-qualified type, top level cv-qulifiers of A's type are ignored
  
看例子，具体的推导过程写在注释里。
```cpp
template <class T> void f(T);
 
int a[3];
f(a);     // P = T,  
	  // A = int[3], after adjustment, transformed A is int*. (Rule 1.1)
	  // T -> int*   calls: f<int*>()

void g(int);
f(g); // P = T, 
      // A = void(int), adjusted to void(*)(int).  (Rule 1.1)
      // T -> void(*)(int)  calls:f<void(*)(int) >()

const int b = 13;
f(b); // P = T, 
	  // A = const int, transformed A is int. (Rule 1.3)
	  // T -> int  calls:f<int>()

//A is a reference
Foo&& fref = Foo();
f (fref);// P = T, 
	  // A = Foo&&.   'Foo' is used for deduction
	  // T -> Foo.   calls:f<Foo>(Foo)
```

### (Rule 2): if P is cv-qualified type, the top level cv-qualifiers of P's type are ignored for tyep deduction
```cpp
template <class T> void f(const T);
int i = 1;
f (i);   // P = const T,  deduced P is T.
	  // A = int
	  // T -> int  calls:f<int>()
const int j = 1;
f (i);   // P = const T,  deduced P is T.
	  // A = const int, transformed A is int (Rule 1.3)
	  // T -> int  calls:f<int>()
```

### (Rule 3): if P is a reference type, the type referred by P is used for type deduction
这里reference type包括lvalue reference 和 rvalue reference.
```cpp
template <class T> void f ( T&);

const int i = 1;
f (i);   // P = T&,  deduced P is T.  (Rule 3)
	  // A = const int, no adjustment
	  // T -> int  calls:f<const int>(const int&)

template <class T> void g ( T&&);
const int i = 1;
f (i);   // P = T&&,  deduced P is T.  (Rule 3)
	  // A = const int, no adjustment
	  // T -> int  calls:f<const int>(const int&&)

Foo && xvalue() { Foo f; return static_cast<Foo&&>f; }  //funciton call is an xvalue
fun_rvalue_ref_param ( xvalue () );
	//P is 'T&&' type, deduced P is T.
	//A is 'Foo&&' type, (xvalue), transformed A is Foo
	//deduces: T is Foo.
	//calls: fun_rvalue_ref_param<Foo>(Foo &&)
```
接着看另一个接近的case
```cpp
Foo && xvalue() { Foo f; return static_cast<Foo&&>f; }  //funciton call is an xvalue
Foo && ref = xvalue ();
fun_rvalue_ref_param ( ref  );
```
推导结果却不是fun_rvalue_ref_param<Foo>(Foo &&) 了， 虽然 ref  和 xvalue的类型(Type)是一样的，但是推导就是不一样，看下一条规则。 

### (Rule 4) special case:  P is T&&, and A is an lvalue
 > ([N3690][1], 14.8.2.1/3)  if P is rvalue refernce to a cv-unqualified template parameter and the argument is an lvalue, the type "lvalue reference to A" is used in place of A for type deduction (special case)

如果模板参数P是T或者T&, 那么类型推导只需要关注P/A的类型即可。 但是当P 为 T&&时，则还涉及expresion category taxonomy。类型推导不仅仅关注P/A的类型，还要关注A 这个expression的category是lvalue 还是xvalue,  pvalue. 
这里多举几个例子，涉及了几个推导规则。每一个都有详细的推导过程。

```cpp
Foo && rvalue_ref () { return Foo (); }
const Foo&& const_rvalue_ref () { return Foo (); }

//universal
template <typename T> void fun_rvalue_ref_param (T&& val) { };

int main {

	int int_lvalue_arg = 2;
	const int int_const_lvalue_arg = 1;

	int& int_lvalue_ref_arg = int_lvalue_arg;
	const int& int_const_lvalue_ref_arg = int_const_lvalue_arg;

	fun_rvalue_ref_param (int_lvalue_arg);
	//P is 'T&&' type, deduced P is T. (Rule 3)
	//A is 'int' type, lvalue, transformed A is 'int&' (rule 4, special case)
	//calls: fun_rvalue_ref_param <int&> (int&).

	fun_rvalue_ref_param (int_const_lvalue_arg);
	//P is 'T&&' type, deduced P is T.
	//A is 'const int' type, lvalue, (special case), transformed A is 'const int &'
	//calls: fun_rvalue_ref_param <const int &> (const int &)

	fun_rvalue_ref_param (int_lvalue_ref_arg);
	//P is 'T&&' type, deduced P is T.
	//A is 'int&' type, lvalue, (special case)
	//calls: fun_rvalue_ref_param <int &> (int &)


	fun_rvalue_ref_param (int_const_lvalue_ref_arg);
	//P is 'T&&' type, deduced P is T.
	//A is 'const int&' type, lvalue, (special case)
	//calls: fun_rvalue_ref_param <const int &> (const int &)

	fun_rvalue_ref_param (3);  //rvalue.  build in
	//P is 'T&&' type, deduced P is T.
	//A is 'int', rvalue, no adjustment. transformed A is int
	//deduces: T is int.
	//calls: fun_rvalue_ref_param<int>(int &&)

	fun_rvalue_ref_param ( Foo() );
	//P is 'T&&' type, deduced P is T.
	//A is 'Foo' type, rvalue (pvalue), no adjustment. transformed A is Foo
	//deduces: T is Foo.
	//calls: fun_rvalue_ref_param<Foo>(Foo &&)

	Foo&& foo_rvalue_ref_arg = Foo();
	const Foo&& foo_const_rvalue_ref_arg = Foo();

	fun_rvalue_ref_param (foo_rvalue_ref_arg);
	//A's type is rvalue reference, it is a named reference, so it is an lvalue. special case!
	//calls: fun_rvalue_ref_param<Foo&>(Foo &)

	fun_rvalue_ref_param (foo_const_rvalue_ref_arg);
	//A is lvalue,  special case
	//calls: fun_rvalue_ref_param<Foo const &>(Foo const &)


	fun_rvalue_ref_param (rvalue_ref());
	//P is 'T&&' type, deduced P is T.
	//A is 'Foo&&' type, (xvalue, not a lvalue), transformed A is Foo
	//deduces: T is Foo.
	//calls: fun_rvalue_ref_param<Foo>(Foo &&)


	fun_rvalue_ref_param (const_rvalue_ref());
	//P is 'T&&' type, deduced P is T.
	//A is 'const Foo&&' type, (xvalue, not a lvalue), transformed A is const Foo
	//deduces: T is Foo.
	//calls: fun_rvalue_ref_param<Foo>(Foo &&)

	return 0;
 }

```

### (Rule 5) 几个deduced A和tranformed A不一致的规则

>(rule 5): In general, the deduction process attempts to find template argument values that will make the deduced A
identical to A (after the type A is transformed as described above). However, there are three cases that allow
a difference:
—	(5.1) — If the original P is a reference type, the deduced A (i.e., the type referred to by the reference) can be more cv-qualified than the transformed A.
—	(5.2) — The transformed A can be another pointer or pointer to member type that can be converted to the
 			deduced A via a qualification conversion (4.4).
—	(5.3) — If P is a class and P has the form simple-template-id, then the transformed A can be a derived class of
 			the deduced A. Likewise, if P is a pointer to a class of the form simple-template-id, the transformed A
 			can be a pointer to a derived class pointed to by the deduced A.

(5.1)
如果P是reference type, deduced A 可以比transformed A多出 cv-qualified。
```cpp
template<typename T> void f1(const T& t);
bool a = false;
f1(a); // P=const T&, adjusted to const T, A=bool, 
       // deduced T = bool, deduced A = const bool
       // deduced A is more cv-qualified than A
```
(5.2)
和(5.1)类似，是指针或者类的成员的情况。
```cpp
template<typename T> void f(const T*);
int* p;
f(p); // P=T, A=int*
      // deduces T=int, deduced A = const int*
      // qualification conversion applies (from int* to const int*)
```

(5.3)
如果P是simple-template-id的形式(template-name < template-argument-listopt>)，可以有类继承关系。

```cpp
template <class T> struct B { };
template <class T> struct D : public B<T> {};
 
template <class T> void f(B<T>&){}
 
void f() {
    D<int> d;
    f(d);  // P = B<T>&, adjusted P = B<T> (a simple-template-id)
           // A = D<int>
           // deduced T = int, deduced A = B<int>
           // A is derived from deduced A
}
```
<br/>
## Non-deduced contexts
如下情况，P不参与template argument deduction。

1) The nested-name-specifier (everything to the left of the scope resolution operator ::) of a type that was specified using a qualified-id.
在 操作符::左边的T不参与deduction，这个规则是理解std::forward实现的关键点之一。
```cpp
// the identity template, often used to exclude specific arguments from deduction
template <typename T> struct identity { typedef T type; };
 
template <typename T>
void bad(std::vector<T> x, T value = 1);
 
template <typename T>
void good(std::vector<T> x, typename identity<T>::type value = 1);
 
std::vector<std::complex<double>> x;
bad(x, 1.2); // P1 = std::vector<T>, A1 = std::vector<std::complex<double>>
             // P1/A1 deduction determines T = std::complex<double>
             // P2 = T, A2 = double
             // P2/A2 deduction determines T = double -- Error
good(x, 1.2); // P1/A1 deduces T = std::complex<double>
              // P2 = identity<T>::type, T is to the left of ::, non-deduced
```

2) The expression of a decltype-specifier.
```cpp
template<typename T> void f(decltype(*std::declval<T>()) arg);
int n; f<int*>(n); // P = decltype(*declval<T>()), A=int. T is not deducible (since C++14)
```

3) A template parameter used in the parameter type of a function parameter that has a default argument that is being used in the call for which argument deduction is being done
```cpp
template<typename T, typename F>
void f(const std::vector<T> &arr, const F& comp = std::less<T>());
std::vector<std::string> arr(3);
f(arr); // P1 = const std::vector<T> &, A1=std::vector<std::string> lvalue,
        // P1/A1 deduces T = std::string
        // P2 = non-deduced context for F (template parameter) used in the
        // parameter type const F& of the function parameter comp,
        // that has a default argument that is being used in the call f(arr)
```
还有其他case，不太常用，这里不提。可参考[文档][2]

注：
1. [标准链接][1]是草稿,只有微小差别，可以去isocpp.org找最新版.

[1]: http://isocpp.org/files/papers/N3690.pdf 
[2]: http://en.cppreference.com/w/cpp/language/template_argument_deduction
[3]: http://kexianda.info/2014/11/20/cpp_basic_concepts
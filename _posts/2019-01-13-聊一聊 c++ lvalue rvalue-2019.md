---
layout:     post
title:      聊一聊 c++ lvalue rvalue
subtitle:   lvalue & rvalue
date:       2019-01-13
author:     唐瑞甫
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - c++
---  

##聊一聊 c++ lvalue rvalue
先来看一段简单的代码

```
int *a;
const int* &p = a;
```
乍一看，好像没什么问题。  
***BUT*** 嘿嘿  
编译器报错了 

```
error: invalid initialization of non-const reference of type ‘const int*&’ from an rvalue of type ‘const int*’ 
```
  
要弄清楚这个问题，先要弄清楚什么是**左值**，什么是**右值**，以及(const)**左值**跟**右值**是如何转换的。  
  
先看左值 (**lvalue**)

> An lvalue is an expression that yields an object reference, such as a variable name, an array subscript reference, a dereferenced pointer, or a function call that returns a reference. An lvalue always has a defined region of storage, so you can take its address. 
  
在看右值 (**rvalue**)

> An rvalue is an expression that is not an lvalue. Examples of rvalues include literals, the results of most operators, and function calls that return nonreferences. An rvalue does not necessarily have any storage associated with it.

简单来说，**左值可以通过&进行取地址的操作，而右值不可以**。  
  
回到开头的那个问题，我们来看下这个表达式

> const int* &p = a;
  
显然，p是一个int\*的左值引用，而b呢，是一个int\*的左值指针，如果像这样直接赋值，是没有任何问题的。

```
int *a;
int* &p = a;
```
  
  
但是不要忘了，p前面还有一个**const**。  
  
问题就出在这个地方。

来仔细的看下p指针 
> const int\* &p  

由于const在int前端，从语义上来讲，p是一个指向const int\*的 **non-const** 引用。如果将一个int\*类型的左值，直接赋值给一个const int\*的左值，需要做隐式转换。
  
对于隐式转换，有以下一些限制需要注意
>1. An type-qualified rvalue of any type, containing zero or more const or volatile qualifications, can be converted to an rvalue of type-qualified type where the second rvalue contains more const or volatile qualifications than the first rvalue.  
>There is partial ordering of cv-qualifiers by the order of increasing restrictions. The type can be said more or less cv-qualified then:  
>>unqualified < const  
unqualified < volatile  
unqualified < const volatile  
const < const volatile  
volatile < const volatile  

>2. A lvalue of any non-function, non-array type T can be implicitly converted to a rvalue of the same type. If T is a non-class type, this conversion also removes cv-qualifiers.
  
所以此时b是个int\***左值**，需要先通过第2条规则将b转换为一个int\***右值**，然后再通过第1条规则，将b转换成一个const int\***右值**。
既然b现在是右值，那么是可以用来初始化const引用的。但前文已经分析了此时p是一个**non-const** 引用。所以就会触发编辑器报错
**不能用一个const int\*右值来初始化const int\*的non-const引用**

**小结**一下这边文章，文章开头从一个小的问题出发，然后引入了lvalue, rvalue的概念，进而分析了const-qualified lvalue/rvalue之间的一些转换规则。若有理解不对的地方，欢迎及时指正。

其实，文中埋下了一个伏笔，也就是C++ 11中大名鼎鼎的 **rvalue reference**及其**std::move()语义**  
下回见。  

---
  By 唐瑞甫
  2019-01-13 


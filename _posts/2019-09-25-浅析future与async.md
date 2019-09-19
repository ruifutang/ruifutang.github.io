---
layout:     post  
title:      浅析 future 与 async  
subtitle:   多线程还可以这样玩
date:       2019-09-25  
author:     唐瑞甫  
header-img: img/post-bg-coffee.jpeg  
catalog: true  
tags:  
    - c++  

---  

前面两篇文章介绍了如何利用 c++0x 来实现一个线程池，其中涉及到了 thread， future， packaged_task 等的使用，这篇文章选取了之前出现过的 「future」及 未出现过的「async」，来一起探讨下 c++0x 语义下如何更好的 「并发」。  
  
首先还是由一个最简单的例子入手  
  
```cpp

template<typename T>
T myEcho( T param)
{
    return param;
}

int main(int argc, char** argv)
{
 
    int iRes1 = myEcho(1), iRes2 = myEcho(2);

    int iResult = iRes1 + iRes2;
    
    std::cout << "Result is " << iResult << std::endl;
}

```  
  
很显然，这里 myEcho 是可以并发执行的。    
但是直接使用 std::thead 会存在一些问题。  
  
#### std::thread 无法处理返回值  
如果我们利用 thread 来进行并发，只能是利用外部变量将结果保存下来。

```cpp  
template<typename T>
T MyEcho( T param)
{
    return param;
}

template<typename T>
void threadEcho(T param, T& res)
{
    res = MyEcho(param);
}

int main(int argc, char** argv)
{
    int iRes1=0, iRes2=0;
    
    std::thread th1(threadEcho<int>, 1, std::ref(iRes1)),
                th2(threadEcho<int>, 2, std::ref(iRes2));
                
    th1.join();
    th2.join();

    int iResult = iRes1 + iRes2;
    
    std::cout << "Result is " << iResult << std::endl;
}
```  
注意这里的 std::ref 的使用，如果直接使用变量的话编译就会报错,可以参考[C++ Thread taking reference argument failed compile](https://stackoverflow.com/questions/36341724/c-thread-taking-reference-argument-failed-compile)

当然也可以简洁一点

```cpp   

int main(int argc, char** argv)
{
    int iRes1=0, iRes2=0;
    
    auto f1 = [&](int i){iRes1 += i;};
    auto f2 = [&](int i){iRes2 += i;};
    
    std::thread th1(f1, 1),
                th2(f2, 2);
                
    th1.join();
    th2.join();

    int iResult = iRes1 + iRes2;
    
    std::cout << "Result is " << iResult << std::endl;
}
  
```
这里你可能有个小疑问了，难道 lambda 不支持 template 吗？可以参考这个 [Can lambda functions be templated?](https://stackoverflow.com/questions/3575901/can-lambda-functions-be-templated)  
  
尽管 c++0x 不直接支持多态 lambda (polymorphic lambda)，但是还是可以利用 decltype 来实现
  
```c++
template<typename T>
T foo( T& res, T t)
{
    auto f = [&](decltype(t) t){ res += t;};
    f(t);
}

int main(int argc, char** argv)
{
    int iRes1=0, iRes2=0;
    
    std::thread th1(foo<int>, std::ref(iRes1), 1),
                th2(foo<int>, std::ref(iRes2), 2);
                
    th1.join();
    th2.join();

    int iResult = iRes1 + iRes2;
    
    std::cout << "Result is " << iResult << std::endl;
}  
```
  
即便如此依然改变不了 std::thread 无法 「直接」 返回结果的现实。为此刚好引出今天要介绍的 async 跟 future，可以参考[What is std::promise?](https://stackoverflow.com/questions/11004273/what-is-stdpromise)。  
  
> The asynchronous provider is what initially creates the shared state that a future refers to. std::promise is one type of asynchronous provider, std::packaged_task is another, and the internal detail of std::async is another. Each of those can create a shared state and give you a std::future that shares that state, and can make the state ready.

 
回到返回值上，C++0x 提供了「异步」处理机制可以解决这个问题。
  
```cpp  
template<typename T>
T myEcho( T param)
{
    return param;
}

int main(int argc, char** argv)
{
 
   std::future<int> futureRes1 = std::async(std::launch::async, myEcho<int>, 1),
                    futureRes2 = std::async(std::launch::async, myEcho<int>, 2);

    int iResult = futureRes1.get() + futureRes2.get();
    
    std::cout << "Result is " << iResult << std::endl;
}
```  
  
相对于最开始的版本，只是通过 async 与 future 语义将其异步化，代码结构几乎不变。需要注意的是这里选用的是 std::launch::async，当然你也可以使用 std::launch::deferred，从名字就可以看出来采用的是 lazy evaluation 的模式，直到 get() 被调用时才会真正地去计算。  
  
当然这里也可以用上面提到的 promise 来实现，这里就不做过多的展示了，原理都是类似的。  
  
#### std::thread 无法捕获异常  
这里对上面用 std::thread 实现的程序做了下简单的处理，当判断类型为 int 时直接抛出异常  

```cpp
template<typename T>
T MyEcho( T param)
{
    if(typeid(T) == typeid(int))
    {
        throw std::runtime_error("oh int");
    }
    return param;
}

template<typename T>
void threadEcho(T param, T& res)
{
    res = MyEcho(param);
}

int main(int argc, char** argv)
{
    try
    {
        int iRes1=0, iRes2=0;
    
        std::thread th1(threadEcho<int>, 1, std::ref(iRes1)),
                th2(threadEcho<int>, 2, std::ref(iRes2));
                
        th1.join();
        th2.join();

        int iResult = iRes1 + iRes2;
    
        std::cout << "Result is " << iResult << std::endl;
    }
    catch ( std::exception& exp)
    {
        std::cerr << exp.what() << std::endl;
    }
}
```
可以看到没有办法捕获到异常。  
换成 future 就可以直接在主线程捕获了。  

```cpp
template<typename T>
T myEcho( T param)
{
    if(typeid(T) == typeid(int))
    {
        throw std::runtime_error("too long");
    }
    return param;
}

int main(int argc, char** argv)
{
    try
    {
        std::future<int> futureRes1 = std::async(std::launch::async, myEcho<int>, 1),
                        futureRes2 = std::async(std::launch::async, myEcho<int>, 2);

        int iResult = futureRes1.get() + futureRes2.get();
    
        std::cout << "Result is " << iResult << std::endl;
    }
    catch ( std::exception& exp)
    {
        std::cerr << exp.what() << std::endl;
    }
}
```
  
#### 小结  
c++0x 对于线程的异步操作提供了大量新的特性来支持，使得在并发的场景中可以更灵活的进行操作。但是不要忘了并发场景中还需要考虑到之前提到的 「memory fense」哦。  
  
期待有一天 c++ 能够将协程在 language 层面进行支持。
  
『明天会更好，不是吗？』  
  
---
  By 唐瑞甫  
  2019-09-25


---
layout:     post
title:      图解epoll的ET模式与LT模式
subtitle:   ET or LT 
date:       2019-01-26
author:     唐瑞甫
header-img: img/post-bg-coffee.jpeg
catalog: true
tags: 
    - epoll


---  

上一篇利用**epoll**实现了一个简单的reactor的demo。众所周知,epoll有两种不同的模式，分别是ET模式跟LT模式。

> 需要注意的是，只有ET模式是epoll独有的，而select()跟poll()都只支持LT模式

### 几种场景  

#### ET  
那先来看一下ET模式下被触发的情况，依旧读跟写的场景分开。  
先看ET读的情况。  
  

1. **当buffer由不可读(空)的状态变成可读(有待读数据)状态的时候**。
![epoll_et1](https://wx1.sinaimg.cn/mw690/9a30a1bagy1fzjupvjfkpj20sg0lc0tm.jpg)   
  
  

2. **当buffer中的待读取的数据增多的时候**。
![epoll_et2](https://wx3.sinaimg.cn/mw690/9a30a1bagy1fzjuq457n3j20sg0lcq40.jpg)   
  
  
3. **当buffer中有数据可读,且对fd调用epoll_mod IN事件的时候**。
  
  
那这几种情况应该怎么验证呢？别急，先接着说ET写触发的情况，跟读的情况类似。  

1. **当buffer由不可写(满)的状态变程可写(有可写空间)状态的时候**。
![epoll_et3](https://wx2.sinaimg.cn/mw690/9a30a1bagy1fzjuq4869fj20sg0lc3zd.jpg)   
  
  
2. **当buffer中待写的数据减少的时候**。
![epoll_et4](https://wx2.sinaimg.cn/mw690/9a30a1bagy1fzjuq4bzjuj20sg0lc0to.jpg)   
  
  
3. **当buffer中有空间可写，且对fd调用epoll_mod OUT事件的时候**。  
  
  
  
#### LT
ET的几种场景已经介绍完了，下面开始介绍LT触发的场景。相对于ET而言，LT触发的场景更“通用”一些。**所有触发ET模式的场景，都能够触发LT模式**。  
下面两种更常见的情况，则只会触发LT模式。  

对于LT读而言  
- 当buffer中待读数据减少，而又未完全读取完的时候，**只**会触发LT模式。
![epoll_lt1](https://wx4.sinaimg.cn/mw690/9a30a1bagy1fzjuq4dc13j20sg0lcdgv.jpg)   
  
  
对于LT写而言  
- 当buffer可写空间减少，而空间未完全被占满的时候，**只**会触发LT模式。
![epoll_lt1](https://wx4.sinaimg.cn/mw690/9a30a1bagy1fzjwek2y5gj20sg0lcab1.jpg) 


### 一个实验
可以做个简单的实验来验证下上面说的几种情况。  


```

# define MAX_EVENTS 10
int main()
{
   
    struct epoll_event event, events[MAX_EVENTS];
    int epoll_fd = epoll_create1(0);
    event.data.fd=STDIN_FILENO;
    event.events=EPOLLIN|EPOLLET;//ET模式
    epoll_ctl(epoll_fd,EPOLL_CTL_ADD,STDIN_FILENO,&event);
    int event_count;
    while(1)
    {
       event_count = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
       for(int i=0;i<event_count;i++)
       {
         if(events[i].data.fd == STDIN_FILENO)
         {
            std::cout<<" epoll EEEEEEEEEEEEEEEET "<<std::endl;
         }
       }
     }
}
```

这段代码的运行结果如下图所示。
![epoll_exp1](https://wx2.sinaimg.cn/mw690/9a30a1bagy1fzkg4d5yiqj20az04st8i.jpg)   
  
每当键盘有输入动作是，就会触发那条std:cout语句。这是因为用户每输入一组字符，都会导致buffer中的可读数据增多，进而导致fd状态的改变，对应于上文ET读模式下的**第二种**情况，会唤醒ET模式下的epoll_wait().  
  
这里简单的将ET模式切换成LT模式，其他不变，对应的代码如下。  

```
# define MAX_EVENTS 10
int main()
{
   
    struct epoll_event event, events[MAX_EVENTS];
    int epoll_fd = epoll_create1(0);
    event.data.fd=STDIN_FILENO;
    event.events=EPOLLIN;//默认LT模式
    epoll_ctl(epoll_fd,EPOLL_CTL_ADD,STDIN_FILENO,&event);
    int event_count;
    while(1)
    {
       event_count =epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
       for(int i=0;i<event_count;i++)
       {
         if(events[i].data.fd == STDIN_FILENO)
          {
            std::cout<<" epoll LLLLLLLLLLLLLLLLLT "<<std::endl;
          }
       }
    }
}

```
  
对应结果如下  
![epoll_exp2](https://wx4.sinaimg.cn/mw690/9a30a1bagy1fzkg4dgis0j20fg04pdfo.jpg)   
  
只要键盘有任何输入，则会出现上图刷屏的情况，也就是程序陷入了死循环。那么为什么改成LT模式下就会出现这个情况呢？这是因为只要输入任意数据后，数据被传入buffer，所以LT模式下每次都会由于buffer有数据可读而触发epoll_wait(),所以会陷入死循环中。  
  
那ET模式下是否会出现这种情况呢？答案是肯定的，而且方法有按照上面的场景不止一种，这里以上文ET读模式下的第三个场景来进行说明。

```
# define MAX_EVENTS 10
int main()
{
   
    struct epoll_event event, events[MAX_EVENTS];
    int epoll_fd = epoll_create1(0);
    event.data.fd=STDIN_FILENO;
    event.events=EPOLLIN|EPOLLET;//LT模式
    epoll_ctl(epoll_fd,EPOLL_CTL_ADD,STDIN_FILENO,&event);
    int event_count;
    while(1)
    {
       event_count =epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
       for(int i=0;i<event_count;i++)
       {
         if(events[i].data.fd == STDIN_FILENO)
          {
            std::cout<<" epoll EEEEEEEEEEEEEEEET "<<std::endl;
  		  event.data.fd=STDIN_FILENO;
            event.events=EPOLLIN|EPOLLET;//使用ET模式
            epoll_ctl(epoll_fd,EPOLL_CTL_MOD,STDIN_FILENO,&event);//重新MOD事件
          }
       }
    }
}

```
  
结果如下图所示。  

![epoll_exp1](https://wx3.sinaimg.cn/mw690/9a30a1bagy1fzkg4d94cmj20ej04at8j.jpg)   
  
有感兴趣的可以将这段代码里面的**EPOLL\_CTL\_MOD**改成**EPOLL\_CTL\_ADD** 试下， 看会出现什么结果。
  
**ET/LT**写模式下的验证方法类似，就不演示了。  

### 小结
本文分别对**ET/LT**模式下读跟写被触发的场景进行了分析，并且给出了代码示例可以在linux下进行验证。之前看了一小部分epoll源码，推荐大家可以去拜读下，了解了底层原理再来看上层的这些场景也可以帮助理解底层的实现。



---
  By 唐瑞甫
  2019-01-26


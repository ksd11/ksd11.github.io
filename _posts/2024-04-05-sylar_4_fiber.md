---
layout: post
title: syalr-4-fiber
author: Qiang Liu
# date: 2024-04-05
toc:  true
tag: [sylar]
---
对sylar的协程模块进行说明

## 基础知识

这一章介绍sylar任务调度模块中的协程子模块。sylar是支持多线程多协程的服务器框架，什么是协程？sylar为什么要采用多线程多协程的设计呢？

### 什么是协程？

对协程概念的理解可以对比线程，通用的说法是协程是⼀种轻量级线程，⽤户态线程。

协程的本质就是函数和函数运行状态的组合 。协程和函数的不同之处是，函数⼀旦被调用，只能从头开始执行，直到函数执行结束退出，而协程则可以执行到⼀半就退出（称为yield），但此时协程并未真正结束，只是暂时让出CPU执行权，在后面适当的时机协程可以重新恢复运行（称为resume），在这段时间里其他的协程可以获得CPU并运⾏，所以协程被描述称为轻量级线程。

### sylar为什么要采用多线程多协程的设计呢？

协程被广泛应用于异步编程，即程序在等待IO或其他事件时可以继续执行其他任务，由于其轻量级的特性，这使得协程之间的切换代价非常低，使得采用协程的应用具有极高的性能。

在服务器框架设计中，我们要处理各种IO事件，如socket的accpet，read，write等事件。这类事件通常会阻塞线程，而利用协程，采用异步编程的模型，可以极大提高系统性能，并简化上层服务设计。

### 什么是对称协程与非对称协程？

两者的本质区别在于**调度权的转移**，对称协程中，在每个协程都可以切换到任意一个其他的协程，而在非对称协程中，协程之间的切换具有类似函数调用的关系，若协程A切换到协程B，那么协程B只能Yield到协程A。

Go语言的协程(go coroutine)是对称协程，而sylar的协程和腾讯的libco协程库采用的是非对称协程。

syalr的协程是非对称协程中一种非常简单的实现，存在着主协程和子协程的概念，协程的切换只发生在主协程和子协程，系统只存在一个主协程，而子协程之间不能直接切换。他们之间的切换逻辑如下如所示。![image-20240406010414510](\assets\sylar\image-20240406010414510.png)

> 协程是一个抽象的概念，具体到代码中，协程只是一段上下文信息，比如执行的地址，寄存器值等，我们用一个结构体Fiber来保存这些信息，因此可以说，每个Fiber对象就对应一个协程。这是理解代码的关键！



## 使用

协程的使用非常简单，我们通过`new Fiber(cb)`创建一个协程，然后执行`Resume`/`Yield`切换协程即可

> 对sylar的fiber相关源码做了改动，使得其更容易理解，并且减少了代码冗余。比如SwapIn/SwapOut函数替换成更易理解的Resume/Yield，call/back相关函数与Resume/Yield合并等。相关代码在：https://github.com/ksd11/sylar-simple

```c++
#include "sylar/sylar.h"

sylar::Logger::ptr g_logger = SYLAR_LOG_ROOT();

void run_in_fiber() {
    SYLAR_LOG_INFO(g_logger) << "run_in_fiber begin";
    // Yield和YieldToHold的区别是，前者是成员方法，后者为静态方法，后者会获取当前协程，然后调用Yield
    sylar::Fiber::YieldToHold(); // -----------------> 2
    SYLAR_LOG_INFO(g_logger) << "run_in_fiber end";
    sylar::Fiber::YieldToHold(); // -----------------> 4
    
    // -> MainFunc - Yield()  // --------------------> 6
}

// 每个线程运行此函数
void test_fiber() {
    SYLAR_LOG_INFO(g_logger) << "main begin -1";
    {
        sylar::Fiber::GetThis(); // 设置主协程
        SYLAR_LOG_INFO(g_logger) << "main begin";
        sylar::Fiber::ptr fiber(new sylar::Fiber(run_in_fiber, 0, true)); // 创建一个协程，运行run_in_fiber
        fiber->Resume(); // --------------------------> 1
        SYLAR_LOG_INFO(g_logger) << "main after swapIn";
        fiber->Resume(); // --------------------------> 3
        SYLAR_LOG_INFO(g_logger) << "main after end";
        fiber->Resume(); // --------------------------> 5
    }
    SYLAR_LOG_INFO(g_logger) << "main after end2"; // 7
}

int main(int argc, char** argv) {
    sylar::Thread::SetName("main");

    std::vector<sylar::Thread::ptr> thrs;
    for(int i = 0; i < 3; ++i) {
        thrs.push_back(sylar::Thread::ptr(
                    new sylar::Thread(&test_fiber, "name_" + std::to_string(i))));
    }
    for(auto i : thrs) {
        i->join();
    }
    return 0;
}
```





## 实现

TODO.

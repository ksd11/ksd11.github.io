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



### 协程切换

sylar的协程实现依赖于Linux下的ucontext族函数

```c++
#include <ucontext.h>


int getcontext(ucontext_t *ucp); // 获取当前上下文
int setcontext(const ucontext_t *ucp); // 设置当前上下文
void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...); // 修改ucp上下文
int swapcontext(ucontext_t *oucp, const ucontext_t *ucp); // 保存当前上下文到oucp，然后切换到ucp

// ucontext_t保存协程上下文信息
typedef struct ucontext_t {
    struct ucontext_t *uc_link; // 当前协程终止后将resume的下一个协程
    sigset_t          uc_sigmask; // 当前协程屏蔽的信号
    stack_t           uc_stack; // 当前协程的栈指针
    mcontext_t        uc_mcontext; // 保存的上下文
               ...
} ucontext_t;
```

通过上述接口，我们就能够很方便的创建、保存、恢复一段上下文（协程）。[这里](https://developer.aliyun.com/article/52886) 给出了如何使用这些接口的示例，强烈建议先通读一遍。后文将假设读者有使用这些接口的基础。

> 协程是一个抽象的概念，具体到代码中，协程只是一段上下文信息，比如执行的地址，寄存器值等。在sylar中，我们用一个结构体Fiber来保存这些信息，因此可以说，每个Fiber对象就对应一个协程。这是理解代码的关键！





## 使用

协程的使用非常简单，我们通过`new Fiber(cb)`创建一个协程，然后执行`Resume`/`Yield`切换协程即可

> 我对sylar的fiber相关源码做了改动，使得其更容易理解，并且减少了代码冗余。比如SwapIn/SwapOut函数替换成更易理解的Resume/Yield，call/back相关函数与Resume/Yield合并等。[相关代码在这里](https://github.com/ksd11/sylar-simple)

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

### 协程状态

sylar协程一共有5种状态：INIT初始状态、EXEC执行状态、HOLD暂停状态、TERM终止状态、EXCEPT异常状态。他们之间的转换关系如下图所示。

![image-20240406111440684](\assets\sylar\image-20240406111440684.png)

协程刚开始被创建时处于INIT状态，当他执行Resume后，就处于EXEC执行状态；中途可能由于IO事件，会主动让出CPU（通过hook，后面介绍），Yield后，协程处于HOLD状态；当IO事件完成，协程通过Resume恢复运行，此时协程又处于EXEC状态；协程运行完毕，正常退出时，会处于TERM状态；而发生异常时，会进入EXCEPT异常状态。



### 协程上下文

每个线程拥有两个全局变量，分别保存了当前协程信息和主协程信息。回忆一下，YieldToHold静态方法就是通过t_fiber获取当前协程的信息，然后调用Yield方法切换协程。

```c++
static thread_local Fiber *t_fiber = nullptr;           // 当前协程
static thread_local Fiber::ptr t_threadFiber = nullptr; // 主协程
```



### 协程初始化

值得注意的是每个线程刚开始执行时都需要调用GetThis静态方法设置主协程。

```c++
// -------- 主协程 ----------
// GetThis() -> Fiber::Fiber()
Fiber::ptr Fiber::GetThis() {
    if (t_fiber) {
        return t_fiber->shared_from_this();
    }
    // 要是t_fiber不存在，则证明主协程还未初始化，因此需要初始化主协程
    Fiber::ptr main_fiber(new Fiber);
    SYLAR_ASSERT(t_fiber == main_fiber.get());
    t_threadFiber = main_fiber; // 设置主协程
    return t_fiber->shared_from_this();
}

Fiber::Fiber() {
    m_state = EXEC; // 标记协程在运行
    SetThis(this);  // 设置当前协程为自己
    if (getcontext(&m_ctx)) {
        SYLAR_ASSERT2(false, "getcontext");
    }
    ++s_fiber_count;
}

// -------- 子协程 -----------
// 只要工作就是为协程分配栈以及设置callback函数
Fiber::Fiber(std::function<void()> cb
             ,size_t stacksize
             ,bool not_run_in_scheduler)
    : m_id(++s_fiber_id)
    ,m_cb(cb)
    ,m_run_in_scheduler(!not_run_in_scheduler) {
    
    ++s_fiber_count;
    m_stacksize = stacksize ? stacksize : g_fiber_stack_size->getValue();
	
    // 分配栈
    m_stack = StackAllocator::Alloc(m_stacksize);
    if (getcontext(&m_ctx)) {
        SYLAR_ASSERT2(false, "getcontext");
    }
    m_ctx.uc_link = nullptr;
    m_ctx.uc_stack.ss_sp = m_stack;
    m_ctx.uc_stack.ss_size = m_stacksize;
	
    // 设置callback
    makecontext(&m_ctx, &Fiber::MainFunc, 0);
}

```



### 协程切换

协程切换主要是调用swapcontext进行，里面涉及到调度器协程或主协程和当前协程的切换问题，这将在scheduler节介绍，读者有点印象即可。

```c++
void Fiber::Resume() {
    SetThis(this);
    SYLAR_ASSERT(m_state != EXEC);
    m_state = EXEC;
    // 若协程被调度器管理，应该和调度器协程切换，scheduler节介绍
    if (m_run_in_scheduler) {
        if (swapcontext(&Scheduler::GetMainFiber()->m_ctx, &m_ctx)) {
            SYLAR_ASSERT2(false, "swapcontext");
        }
    } else {
    // 协程没被调度器管理，和主协程切换
        if (swapcontext(&t_threadFiber->m_ctx, &m_ctx)) {
            SYLAR_ASSERT2(false, "swapcontext");
        }
    }
}

void Fiber::Yield() {
    SYLAR_ASSERT(!(m_state == INIT));
    if (m_state == EXEC)
        m_state = HOLD;
	
    // 若协程被调度器管理，应该和调度器协程切换，scheduler节介绍
    if (m_run_in_scheduler) {
        SetThis(Scheduler::GetMainFiber());
        if (swapcontext(&m_ctx, &Scheduler::GetMainFiber()->m_ctx)) {
            SYLAR_ASSERT2(false, "swapcontext");
        }
    } else {
    // 协程没被调度器管理，和主协程切换
        SetThis(t_threadFiber.get());
        if (swapcontext(&m_ctx, &t_threadFiber->m_ctx)) {
            SYLAR_ASSERT2(false, "swapcontext");
        }
    }
}

// 子协程创建时上下文设置为此函数
void Fiber::MainFunc() {
    Fiber::ptr cur = GetThis();
    SYLAR_ASSERT(cur);
    try {
        cur->m_cb(); // 然后才真正调用提供的callback
        cur->m_cb = nullptr;
        cur->m_state = TERM; // 返回代表协程结束，设置TERM状态
    } catch (std::exception &ex) {
        cur->m_state = EXCEPT;
        ...
    } catch (...) {
        cur->m_state = EXCEPT;
        ...
    }
	
    // 为什么要裸指针调用Yield?
    auto raw_ptr = cur.get();
    cur.reset();
    raw_ptr->Yield();

    SYLAR_ASSERT2(false,
                  "never reach fiber_id=" + std::to_string(raw_ptr->getId()));
}
```

54行提出了一个问题，为什么要用裸指针调用Yield，而不能直接通过`cur->Yield()`调用？

这是因为协程结束后，将不会再返回，若是一直持有当前协程的智能指针，内存将永远无法被释放，造成内存泄漏。读者可以尝试注释55-60行，然后添加`cur->Yield()`到函数末尾，然后重新编译运行`test_fiber.cc`示例，就能看到有的协程没有正常执行析构函数。



### 其他

#### reset

reset方法可以在协程退出时，重用协程的栈等资源，实现方式就是替换协程的cb函数，然后重新初始化栈指针，设置上下文为MainFunc函数即可。

这里的关键是协程只有在初始化或者退出的状态时才能reset，否则将造成混乱。

```c++
void Fiber::reset(std::function<void()> cb) {
    SYLAR_ASSERT(m_stack);
    SYLAR_ASSERT(m_state == TERM || m_state == EXCEPT || m_state == INIT);
    m_cb = cb;
    if (getcontext(&m_ctx)) {
        SYLAR_ASSERT2(false, "getcontext");
    }

    m_ctx.uc_link = nullptr;
    m_ctx.uc_stack.ss_sp = m_stack;
    m_ctx.uc_stack.ss_size = m_stacksize;

    makecontext(&m_ctx, &Fiber::MainFunc, 0);
    m_state = INIT;
}
```




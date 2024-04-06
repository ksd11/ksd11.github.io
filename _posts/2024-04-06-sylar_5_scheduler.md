---
layout: post
title: syalr-5-scheduler
author: Qiang Liu
# date: 2024-04-06
toc:  true
tag: [sylar]
---
对sylar的协程调度模块进行说明

我们之前想要执行任务时，需要手动new一个协程，然后调用其resume/yield方法，来在协程之间切换。这太繁琐了，如果我们只指定需要完成什么任务，有个调度器自动帮我们new一个协程，并在合适的时机调度协程运行，那该多好。这就是协程调度模块scheduler的意义。

sylar的写成调度器支持多线程多协程，也就是说一个调度器可以管理多个线程，而每个线程都可以运行多个协程（串行执行，而不是并行）。任务就是协程，线程不断地从任务队列中取出任务，然后执行，循环往复，直到没有任务并且调度器停止运行则终止。

## 使用

使用比较简单，创建一个Scheduler对象，通过`start()`方法启动，然后调用其`schedule(FiberOrCb fc)`方法添加任务即可，如果任务全部添加完毕，可通过`stop()`方法关闭调度器，调度器会阻塞直到所有任务完成。

```c++
#include "sylar/sylar.h"

static sylar::Logger::ptr g_logger = SYLAR_LOG_ROOT();

void test_fiber() {
    static int s_count = 5;
	SYLAR_LOG_INFO(g_logger) << "test in fiber s_count=" << s_count;
    sleep(1);
    if(--s_count >= 0) {
        // 固定只能当前线程去执行该协程任务
        sylar::Scheduler::GetThis()->schedule(&test_fiber, sylar::GetThreadId());
    }
}

int main(int argc, char** argv) {
    // 参数含义：指定创建3个线程，将caller线程作为调度线程，线程名字
    sylar::Scheduler sc(3, true, "worker");
    // 创建线程
    sc.start();
    // 将任务加入到任务队列
    sc.schedule(&test_fiber);
    // 停止调度器
    sc.stop();
    return 0;
}

```

比较难理解的是代码第17行，将caller线程作为调度线程是什么意思？

caller线程就是执行scheduler构造的线程，在本场景，就是执行main函数的线程。sylar的调度器是支持多线程的，我们在创建调度器的时候，可以指定需要创建多少个工作线程去执行任务，假设用户指定了k个线程，如果不将caller线程当做是工作线程的话，我们需要新创建k个线程去执行任务，而主线程则会阻塞直到k个线程都完成；如果将caller线程当做是工作线程，我们只需要新创建k-1个线程，caller线程和其他线程一起执行任务，这节省了一个线程的创建开销。



## 实现

### 线程执行流

若将caller线程作为其中一个工作线程，则调用轨迹如下图所示，在调用`scheduler.stop()`时，会检查任务是否完成，没完成的话，会通过Resume从主协程切换到调度协程，并在调度协程中不断执行取任务，做任务的循环。

![image-20240406133146124](\assets\sylar\image-20240406133146124.png)



而对于普通的工作线程，是在`scheduler.start()`函数调用的时候创建的，入口函数即是调度协程（Scheduler::run），工作线程的调度协程亦是其主协程，工作线程也会在调度协程中，不断执行取任务，做任务的循环。

![image-20240406132934646](\assets\sylar\image-20240406132934646.png)



这里的关键点是区分，主协程，调度协程，待调度的任务协程。

- 主协程，对于caller线程来说，从调度器被创建到销毁的执行流所在上下文即是主协程；而对于其他线程，调度协程是其主协程
- 调度线程，对于caller线程来说，调度协程是其子协程
- 待调度的任务协程，这对于所有的线程来说都是一样的，是完成业务逻辑而new出来的任务协程

```c++
// fiber.cc
static thread_local Fiber *t_fiber = nullptr;           // 当前协程
static thread_local Fiber::ptr t_threadFiber = nullptr; // 主协程

// scheduler.cc
static thread_local Fiber *t_scheduler_fiber = nullptr; // 调度协程
```



![image-20240406135419924](\assets\sylar\image-20240406135419924.png)

回想之前fiber章节，Yield和Resume方法都会根据run_in_scheduler参数，来切换不同的上下文，其实这就是在判断是不是在调度协程中，从而来决策是和主协程还是调度协程切换。



### 初始化

初始化就是根据use_caller设置与否，来决定是否将caller线程加入调度

```c++
Scheduler::Scheduler(size_t threads, bool use_caller, const std::string &name,
                     bool hook_enable)
    : m_name(name), hook_enable(hook_enable) {
    SYLAR_ASSERT(threads > 0);

    // 是否将调用者线程作为一个调度线程
    if (use_caller) {
        sylar::Fiber::GetThis(); // 获得主协程
        --threads;               // 需要的线程减少一个

        SYLAR_ASSERT(GetThis() == nullptr);
        caller_thread_id = GetThreadId();
		
        m_rootFiber.reset(new Fiber(std::bind(&Scheduler::run, this), 0, true)); // 创建调度协程
        sylar::Thread::SetName(m_name);

        t_scheduler_fiber = m_rootFiber.get();
        m_rootThread = sylar::GetThreadId();
        m_threadIds.push_back(m_rootThread);
    } else {
        m_rootThread = -1;
    }
    m_threadCount = threads;
}
```



### start

调度器的start方法，是根据用户指定线程个数，创建对应数量的线程，每个线程执行`scheduler::run`方法，即调度协程，从而开始取任务并执行

```c++
void Scheduler::start() {
    MutexType::Lock lock(m_mutex);
    if (!m_stopping) {
        return;
    }
    m_stopping = false;
    SYLAR_ASSERT(m_threads.empty());

    // 创建m_threadCount个线程，并加入线程池
    m_threads.resize(m_threadCount);
    for (size_t i = 0; i < m_threadCount; ++i) {
        m_threads[i].reset(new Thread(std::bind(&Scheduler::run, this),
        m_name + "_" + std::to_string(i)));
        m_threadIds.push_back(m_threads[i]->getId());
    }
    lock.unlock();
}
```



### schedule

schedule是提供给用户添加任务的接口，用户可以指定一个新的fiber或者指定一个函数来创建一个任务协程，被添加的任务协程结构是`FiberAndThread`,在Scheduler中通过一个list成员保存。

```c++
// 封装一个任务协程
// 具有一个fiber指针或者cb函数，thread指定协程被哪个线程执行
struct FiberAndThread{
    Fiber::ptr fiber;
    std::function<void()> cb;
    // 线程id
    int thread;
    ...
};

// 添加一个协程任务
template<class FiberOrCb>
void schedule(FiberOrCb fc, int thread = -1) {
    bool need_tickle = false;
    {
        MutexType::Lock lock(m_mutex);
        need_tickle = scheduleNoLock(fc, thread);
    }

    if(need_tickle) {
    	tickle();
    }
}

// 添加一堆协程任务
template<class InputIterator>
void schedule(InputIterator begin, InputIterator end) {
    bool need_tickle = false;
    {
        MutexType::Lock lock(m_mutex);
        while(begin != end) {
            need_tickle = scheduleNoLock(&*begin, -1) || need_tickle;
            ++begin;
        }
    }
    if(need_tickle) {
        tickle();
    }
}

template<class FiberOrCb>
bool scheduleNoLock(FiberOrCb fc, int thread) {
    bool need_tickle = m_fibers.empty();
    FiberAndThread ft(fc, thread);
    if(ft.fiber || ft.cb) {
        m_fibers.push_back(ft); // 任务添加到队列
    }
    return need_tickle;
}
```





### run

这是Scheduler调度器的核心，每个线程在这里取任务并执行

```c++
void Scheduler::run() {
    SYLAR_LOG_DEBUG(g_logger) << m_name << " run";
    set_hook_enable(hook_enable);
    setThis();

    // 非user_caller线程，每个线程设置自己的主协程
    if (sylar::GetThreadId() != m_rootThread) {
        t_scheduler_fiber = Fiber::GetThis().get();
    }
	
    // 空闲协程，当没有任务时执行
    Fiber::ptr idle_fiber(
        new Fiber(std::bind(&Scheduler::idle, this))); 
    // 任务是cb的形式时，可通过该结构封装成一个协程运行，而协程退出时，该结构可被reset后重新使用
    Fiber::ptr cb_fiber;

    FiberAndThread ft;
    while (true) {
        ft.reset();
        bool tickle_me = false; // 是否需要tick
        {
            MutexType::Lock lock(m_mutex);

            // --- 遍历协程队列,直到找到一个可以运行的任务 ---
            auto it = m_fibers.begin();
            while (it != m_fibers.end()) {
                // 该任务指定了调度线程，但不是当前线程，需跳过
                if (it->thread != -1 && it->thread != sylar::GetThreadId()) {
                    ++it;
                    tickle_me = true;
                    continue;
                }

                SYLAR_ASSERT(it->fiber || it->cb);

                ft = *it;
                m_fibers.erase(it++);
                ++m_activeThreadCount;
                break;
            }
            tickle_me |= it != m_fibers.end();
        }

        if (tickle_me) {
            tickle();
        }
        if (ft.fiber && (ft.fiber->getState() != Fiber::TERM &&
                         ft.fiber->getState() != Fiber::EXCEPT)) {
            // 1. ---- 如果是fiber，并且处于可执行状态 ---
            ft.fiber->Resume();    // 切换协程执行
            --m_activeThreadCount;
            ft.reset();
        } else if (ft.cb) {
            // 2. ---- 如果是函数 -----
            if (cb_fiber) {
                cb_fiber->reset(ft.cb);
            } else {
                cb_fiber.reset(new Fiber(ft.cb));
            }
            cb_fiber->Resume(); // 切换协程执行
            --m_activeThreadCount;

            // 若fiber已经退出了，则该fiber可被重用
            if (cb_fiber->getState() == Fiber::EXCEPT ||
                cb_fiber->getState() == Fiber::TERM) {
                cb_fiber->reset(nullptr);
            } else {
                // 否则，务必要reset
                cb_fiber.reset();
            }
        } else {
            // 3. ---- 如果没有任务，切换到Idle协程 ------
            ++m_idleThreadCount;
            idle_fiber->Resume(); // 切换idle协程执行
            --m_idleThreadCount;
            
            if (idle_fiber->getState() == Fiber::TERM) {
                SYLAR_LOG_INFO(g_logger) << "idle fiber term";
                break;
            }
        }
    }
}

// 目前idle协程啥也不干，只是反复切换到调度协程。（挖坑：未来在io调度协程设计上重写此函数大有用处）
void Scheduler::idle() {
    SYLAR_LOG_INFO(g_logger) << "idle";
    // 若未stopping，则切换到调度协程
    while (!stopping()) {
        sylar::Fiber::YieldToHold();
    }
    // 若stopping，则退出，并设置TERM状态
}

// tick负责通知其他调度协程有任务，在目前什么工作也没做。（挖坑：未来在io调度协程设计上重写此函数大有用处）
void Scheduler::tickle() { SYLAR_LOG_INFO(g_logger) << "tickle"; }
```



### stop

调用stop后caller线程会阻塞直到Scheduler所有任务都完成，若是caller线程也加入了调度，则在这里caller线程主协程也会跳转到调度协程执行。

```c++
void Scheduler::stop() {
    
    // 只允许caller线程调用stop
    assert(caller_thread_id == GetThreadId());
	
    m_stopping = true;
    if (stopping()) {
            return;
    }
	
    // 通知其他线程结束了
    for (size_t i = 0; i < m_threadCount; ++i) {
        tickle();
    }

    if (m_rootFiber) {
        if (!stopping()) {
            m_rootFiber->Resume(); // 主协程 -> 调度协程
        }
    }
    
    // 等待所有的线程退出
    
    // 之所以要swap thrs和m_threads，是为了让Scheduler调用stop后，所有线程的资源都被释放了，而不是等到析构时才释放
    std::vector<Thread::ptr> thrs;
    {
        MutexType::Lock lock(m_mutex);
        thrs.swap(m_threads);
    }
    for (auto &i : thrs) {
        i->join();
    }
}
```


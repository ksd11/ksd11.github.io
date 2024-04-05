---
layout: post
title: syalr-3-thread
author: Qiang Liu
# date: 2024-04-03
toc:  true
tag: [sylar]
---
对sylar的线程模块进行说明



## 使用

sylar的任务调度模块比较复杂，将会分几个章节介绍。这里先介绍线程模块，由于不同线程之间存在着共享资源竞争的问题，我们还需要引入锁、信号量的机制，来保证共享资源访问的一致性。

### Mutex

有多种措施来协调共享资源的访问。如锁和信号量。锁又分为了互斥锁、读写锁、自旋锁。互斥锁在拿不到锁的时候会让出CPU，而自旋锁会使CPU一直空转，直到获取锁。读写锁分为读锁和写锁，在同一时刻可以有多个线程获取读锁，但是同一时刻只能有一个线程获取写锁，并且此时不能有其他线程获取读锁。

另外，还有一种范围锁，这是对其他锁的一种包装，采用了[RAII](https://zh.cppreference.com/w/cpp/language/raii)的机制，即在初始化时获取资源（锁），而在析构时释放资源（锁），这减轻了程序员的负担，避免了忘记初始化或者释放资源。

```c++
// ------ 互斥锁、自旋锁 -------
sylar::Mutex s_mutex; // sylar::Spinlock一样
s_mutex.lock();
// do something
s_mutex.unlock();

// ------ 读写锁 --------
sylar::RWMutex s_mutex;
s_mutex.rdlock(); // 读锁
s_mutex.wrlock(); // 写锁
s_mutex.unlock(); // 释放读锁/写锁

// ------- 范围锁 ---------
sylar::Mutex s_mutex;
sylar::ScopedLockImpl<sylar::Mutex> lock(s_mutex); // 等价于 sylar::Mutex::Lock lock(s_mutex);

sylar::RWMutex s_mutex;
sylar::WriteScopedLockImpl<sylar::RWMutex> lock(s_mutex); // 等价于 sylar::RWMutex::WriteLock lock(s_mutex);
sylar::ReadScopedLockImpl<sylar::RWMutex> lock(s_mutex); // 等价于 sylar::RWMutex::ReadLock lock(s_mutex);

// -------- 信号量 -----------
sylar::Semaphore sem(3);
sem.wait(); // 获取信号量
sem.notify(); // 释放信号量

```



### thread

线程的使用同样非常简单，通过Thread类创建即可

```c++
#include "sylar/sylar.h"
#include <unistd.h>

sylar::Logger::ptr g_logger = SYLAR_LOG_ROOT();

int count = 0;
sylar::Mutex s_mutex;

void fun1() {
    SYLAR_LOG_INFO(g_logger) << "name: " << sylar::Thread::GetName()
                             << " this.name: " << sylar::Thread::GetThis()->getName()
                             << " id: " << sylar::GetThreadId()
                             << " this.id: " << sylar::Thread::GetThis()->getId();

    for(int i = 0; i < 100000; ++i) {
        sylar::Mutex::Lock lock(s_mutex);
        ++count; 
    }
}

int main(int argc, char** argv) {
    std::vector<sylar::Thread::ptr> thrs;
    // 创建2个线程，Thread在new后就开始运行了
    for(int i = 0; i < 2; ++i) {
        sylar::Thread::ptr thr(new sylar::Thread(&fun1, "name_" + std::to_string(i * 2)));
        thrs.push_back(thr);
    }
    // 等待上述线程结束
    for(size_t i = 0; i < thrs.size(); ++i) {
        thrs[i]->join();
    }
    return 0;
}

```



## 实现

mutex和thread实现是对pthread库的一层简单封装，比较简单，不一一介绍。

唯一需要注意的是Thread构造函数中使用了一个信号量，等待Thread初始化完成后才返回，这样可以保证Thread返回后，获取的Thread的信息都是有效的。


---
layout: post
title: syalr-6-iomanager
author: Qiang Liu
# date: 2024-04-06
toc:  true
tag: [sylar]
---
对sylar的IO协程调度模块进行说明

IO事件调度功能对服务器开发至关重要，因为服务器通常要处理大量来自客户端的socket fd，使用IO事件调度可以将开发者从判断socket fd是否可读或可写的工作中解放出来，使得程序员只需要关心socket fd的IO操作，因此，实现IO协程调度意义重大。

IO协程调度的实现依赖IO多路复用技术，我们对感兴趣的事件进行注册，等到事件发生的时候才进行处理。IO协程调度支持为描述符事件注册可读和可写事件的回调函数，当描述符可读或可写时，执行相应的回调函数（协程）。

此外，sylar还实现了毫秒级定时器功能，可以指定事件超时触发一次还是周期触发。定时器功能是实现下一节hook模块所必需的。



## 基础知识

### 为什么采用IO多路复用？

- **提高并发处理能力**：多路复用技术允许服务器使用一个或少量线程来同时处理多个网络连接，这样做的好处是显著减少了线程或进程切换的开销，提高了服务器的整体性能。

- **提高资源利用率**：如果没有IO多路复用，每一个客户端的连接都需要一个线程去处理，并且read/write可能会阻塞，这使得系统开辟了许多线程去处理用户请求但真正工作的线程数并不多，使得资源被白白浪费。而采用IO多路复用，在socket真正需要被处理时才分配资源，大大减低了系统开销。

  

### IO多路复用

Linux系统下实现IO多路复用的技术有select,poll,epoll。前两个都是较老的版本，有文件描述个数限制或是线性遍历慢等问题。后者是最新的，速度也最快，sylar采用的是epoll。这三者的介绍可以[看这里](https://xiaolincoding.com/os/8_network_system/selete_poll_epoll.html#i-o-%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8)。下面对epoll的使用进行简要介绍。

```c++
 #include <sys/epoll.h>

typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};

// 下面是对epoll文件描述符的操作接口
int epoll_create(int size); // 创建一个epoll文件描述符
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); // 添加/删除/修改 epoll文件描述符监听事件
int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout); // 监听事件发生
 
```



使用epoll的工作流程如下所示，我们先通过`epoll_create`创建epoll，然后创建我们感兴趣的事件，并通过`epoll_ctr`添加，最后调用`epoll_wait`等待事件发生。

```c++
// ---- 创建epoll文件描述符 -----
int m_epfd = epoll_create(42); // 参数无意义，大于0即可

// ---- 创建epoll事件：文件描述符，事件 -----
epoll_event event;
memset(&event, 0, sizeof(epoll_event));
event.events = EPOLLIN | EPOLLET;  // 添加监听的事件
event.data.fd = fd;                // 添加监听的描述符
event.data.ptr = (void*)user_data; // 可以传递用户参数

// ---- 添加epoll事件到epoll文件描述符 ----
epoll_ctl(m_epfd, EPOLL_CTL_ADD, fd, &event); // 监听

// 传递给epoll_wait的结构，事件发生时内核填充trigger_events
epoll_event *trigger_events = new epoll_event[MAX_EVNETS]();

while(1) {
    // ---- 阻塞，直到监听事件发生 -----
    int rt = epoll_wait(m_epfd, trigger_events, MAX_EVNETS, (int)timeout);
    for(感兴趣的事件){
        //处理
    }
}
```



### LT和ET模式

epoll 支持两种事件触发模式，分别是边缘触发（edge-triggered，ET）和水平触发（level-triggered，LT）。

- 使用边缘触发模式时，当被监控的 Socket 描述符上有可读事件发生时，**服务器端只会从 epoll_wait 中苏醒一次**，即使进程没有调用 read 函数从内核读取数据，也依然只苏醒一次，因此我们程序要保证一次性将内核缓冲区的数据读取完；
- 使用水平触发模式时，当被监控的 Socket 上有可读事件发生时，**服务器端不断地从 epoll_wait 中苏醒，直到内核缓冲区数据被 read 函数读完才结束**，目的是告诉我们有数据需要读取；



## 使用

IOManager的使用也比较简单，对于普通事件，提供了addEvent，delEvent，cancelEvent，cancelAll接口，可以方便地添加，删除，取消事件；对于定时器事件，提供了addTimer，addConditionTimer等接口，分别是添加定时器和添加条件定时器功能。

### 普通事件

下面的代码我们创建了一个socket，然后通过addEvent为socket添加读写事件。可以看到为同一个sock我们可以方便地添加不同的事件，将call_back替换为不同的register即可，这也是IOManager带给我们的便利，我们只需要处理IO操作即可，而不需要关心socket什么时候可读，什么时候可写了。

```c++
sylar::Logger::ptr g_logger = SYLAR_LOG_ROOT();

static int sock;
void register_1();
void register_2();

// static std::function<void()> call_back = register_1;
static std::function<void()> call_back = register_2;

// 创建一个socket，并添加socket的read,write事件
void test_fiber(){
    SYLAR_LOG_INFO(g_logger) << "test_fiber sock=" << sock;

    sock = socket(AF_INET, SOCK_STREAM, 0);
    fcntl(sock, F_SETFL, O_NONBLOCK);

    sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(80);
    inet_pton(AF_INET, "112.80.248.75", &addr.sin_addr.s_addr);

    if(!connect(sock, (const sockaddr*)&addr, sizeof(addr))) {
    } else if(errno == EINPROGRESS) {
        SYLAR_LOG_INFO(g_logger) << "add event errno=" << errno << " " << strerror(errno);
        
        call_back(); // 执行call_back函数为sock设置read,write事件
    } else {
        SYLAR_LOG_INFO(g_logger) << "else " << errno << " " << strerror(errno);
    }
}

void register_1() {
    // 注册sock读事件
    sylar::IOManager::GetThis()->addEvent(sock, sylar::IOManager::READ, [](){
        SYLAR_LOG_INFO(g_logger) << "read callback";
    });

    // 注册sock写事件
    sylar::IOManager::GetThis()->addEvent(sock, sylar::IOManager::WRITE, [](){
        SYLAR_LOG_INFO(g_logger) << "write callback";
        // 取消sock的读事件（会触发事件一次）
        sylar::IOManager::GetThis()->cancelEvent(sock, sylar::IOManager::READ);
        close(sock);
    });
}

void register_2() {
    // sock读回调
    sylar::IOManager::GetThis()->addEvent(sock, sylar::IOManager::READ, [](){
        SYLAR_LOG_INFO(g_logger) << "read callback";
        char temp[1000];
        int rt = read(sock, temp, 1000);
        if (rt >= 0) {
            std::string ans(temp, rt);
            SYLAR_LOG_INFO(g_logger) << "read:["<< ans << "]";
        } else {
            SYLAR_LOG_INFO(g_logger) << "read rt = " << rt;
        }
    });

    // sock写回调
    sylar::IOManager::GetThis()->addEvent(sock, sylar::IOManager::WRITE, [](){
        SYLAR_LOG_INFO(g_logger) << "write callback";
        int rt = write(sock, "GET / HTTP/1.1\r\ncontent-length: 0\r\n\r\n",38);
        SYLAR_LOG_INFO(g_logger) << "write rt = " << rt;
    });

}

void test1() {
    sylar::IOManager iom(2, false);
    iom.schedule(&test_fiber);
}

int main(int argc, char** argv) {
    test1();
    return 0;
}

```

### 定时器事件

我们通过addTimer添加定时器事件，定时器的单位是毫秒级的，因此s_timer初始时每隔1s触发一次，当`i==3`时，我们更新定时器为2s触发一次。

```c++
sylar::Timer::ptr s_timer;
void test_timer() {
    sylar::IOManager iom(1);
    // 每1s执行一次
    s_timer = iom.addTimer(1000, [](){
        static int i = 0;
        SYLAR_LOG_INFO(g_logger) << "hello timer i=" << i;
        if(++i == 3) {
            s_timer->reset(2000, true);
            //s_timer->cancel();
        }
        if (i == 5) {
            s_timer->cancel();
        }
    }, true);
}

int main(int argc, char** argv) {
    test_timer();
    return 0;
}
```



## 实现

### 普通事件概述

sylar⽀持两类事件，⼀类是可读事件，对应EPOLLIN，⼀类是可写事件，对应EPOLLOUT，sylar的事件枚举值直接继承⾃epoll。

当然epoll本身除了支持了EPOLLIN和EPOLLOUT两类事件外，还支持其他事件，比如EPOLLRDHUP, EPOLLERR,EPOLLHUP等，对于这些事件，sylar的做法是将其进⾏归类，分别对应到EPOLLIN和EPOLLOUT中，也就是所有的事件都可以表示为可读或可写事件，甚⾄有的事件还可以同时表示可读及可写事件，⽐如EPOLLERR事件发⽣时，fd将同时触发可读和可写事件。

对于IO协程调度来说，每次调度都包含⼀个三元组信息，分别是文件描述符-事件类型（可读或可写）-回调函数，调度器记录全部需要调度的三元组信息，其中描述符和事件类型⽤于epoll_wait，回调函数⽤于协程调度。这个三元组信息在源码上通过**FdContext**结构体来存储，在执⾏epoll_wait时通过epoll_event的私有数据指针data.ptr来提取FdContext结构体信息。

IO协程调度器在**idle**时会epoll_wait所有注册的fd，如果有fd满足条件，epoll_wait返回，从私有数据中拿到fd的上下文信息，并且执行其中的回调函数。（实际是idle协程只负责收集所有已触发的fd的回调函数并将其加⼊调度器的任务队列，真正的执⾏时机是idle协程退出后，调度器在下⼀轮调度时执⾏）。

协程调度器支持通过schedule函数添加任务，而IO协程调度器还⽀持**添加，删除，取消事件**。添加的事件将在idle协程中通过epoll_wait监听；取消事件表示不关心某个fd的某个事件了，如果某个fd的可读或可写事件都被取消了，那这个fd会从调度器的epoll_wait中删除。

### FdContext

```c++
// sylar只支持两种事件，其他事件可归类到这两种事件
enum Event {
    /// 无事件
    NONE    = 0x0,
    /// 读事件(EPOLLIN)
    READ    = 0x1,
    /// 写事件(EPOLLOUT)
    WRITE   = 0x4,
};

// 保存文件描述符上下文：监听的事件，事件发生时的回调
struct FdContext {
    typedef Mutex MutexType;
    /**
         * @brief 事件上下文类
         */
    struct EventContext {
        /// 事件执行的调度器
        Scheduler* scheduler = nullptr;
        /// 事件协程
        Fiber::ptr fiber;
        /// 事件的回调函数
        std::function<void()> cb;
    };

    /**
         * @brief 获取事件上下文类
         * @param[in] event 事件类型
         * @return 返回对应事件的上线文
         */
    EventContext& getContext(Event event);

    /**
         * @brief 重置事件上下文
         * @param[in, out] ctx 待重置的上下文类
         */
    void resetContext(EventContext& ctx);

    /**
         * @brief 触发事件
         * @param[in] event 事件类型
         */
    void triggerEvent(Event event);

    /// 读事件上下文
    EventContext read;
    /// 写事件上下文
    EventContext write;
    /// 事件关联的句柄
    int fd = 0;
    /// 当前的事件
    Event events = NONE;
    /// 事件的Mutex
    MutexType mutex;
};

// 根据event类型，返回相应的EventContext
IOManager::FdContext::EventContext &
IOManager::FdContext::getContext(IOManager::Event event) {
    switch (event) {
    case IOManager::READ:
        return read;
    case IOManager::WRITE:
        return write;
    default:
        SYLAR_ASSERT2(false, "getContext");
    }
    throw std::invalid_argument("getContext invalid event");
}

// 重置某个EventContext
void IOManager::FdContext::resetContext(EventContext &ctx) {
    ctx.scheduler = nullptr;
    ctx.fiber.reset();
    ctx.cb = nullptr;
}

// 触发Event，前提是该Event被监听
void IOManager::FdContext::triggerEvent(IOManager::Event event) {

    SYLAR_ASSERT(events & event);
    events = (Event)(events & ~event); // 触发后event剔除
    EventContext &ctx = getContext(event);
    if (ctx.cb) {
        ctx.scheduler->schedule(&ctx.cb);
    } else {
        ctx.scheduler->schedule(&ctx.fiber);
    }
    ctx.scheduler = nullptr;
    return;
}
```



### 初始化

ioManager重载了tickle方法，当有新任务时，会唤醒处于Idle协程的线程去处理任务。这是通过管道实现的，idle协程中执行的epoll_wait方法会监听管道的读端，而tickle函数在检测到存在idle协程时，会往管道写数据，从而唤醒idle协程。

在IOManager构造的时候初始化了管道，然后调用start方法启动线程（继承自Scheduler）

```c++
class IOManager : public Scheduler, public TimerManager{..};

IOManager::IOManager(size_t threads, bool use_caller, const std::string &name,
                     bool hook_enable)
    : Scheduler(threads, use_caller, name, hook_enable) {
    m_epfd = epoll_create(5000);
    SYLAR_ASSERT(m_epfd > 0);

    int rt = pipe(m_tickleFds);
    SYLAR_ASSERT(!rt);

    epoll_event event;
    memset(&event, 0, sizeof(epoll_event));
    event.events = EPOLLIN | EPOLLET;
    event.data.fd = m_tickleFds[0];

    rt = fcntl(m_tickleFds[0], F_SETFL, O_NONBLOCK);
    SYLAR_ASSERT(!rt);

    rt = epoll_ctl(m_epfd, EPOLL_CTL_ADD, m_tickleFds[0],
                   &event); // 监听管道读端
    SYLAR_ASSERT(!rt);

    contextResize(32);

    start(); // IO Manager继承了scheduler，调用start方法启动调度器
}

// m_fdConext的类型为std::vector<FdContext*>,保存了所有fd上下文
void IOManager::contextResize(size_t size) {
    m_fdContexts.resize(size);
    for (size_t i = 0; i < m_fdContexts.size(); ++i) {
        if (!m_fdContexts[i]) {
            m_fdContexts[i] = new FdContext;
            m_fdContexts[i]->fd = i;
        }
    }
}
```



### Event操作

这节包含了对Event相关的操作，包含了addEvent、delEvent、cancelEvent、cancelAll。addEvent添加三元组：fd-event-cb，即fd在发生event时执行cb；delEvent删除fd上的event；cancelEvent取消fd上的event，区别是cancelEvent会触发一次fd上的event；cancelAll取消fd上的所有事件

```c++
int IOManager::addEvent(int fd, Event event, std::function<void()> cb) {
    // 找到fd对应的FdContext
    FdContext *fd_ctx = nullptr;
    RWMutexType::ReadLock lock(m_mutex);
    if ((int)m_fdContexts.size() > fd) {
        fd_ctx = m_fdContexts[fd];
        lock.unlock();
    } else {
        lock.unlock();
        RWMutexType::WriteLock lock2(m_mutex);
        contextResize(fd * 1.5); // 若不存在则扩容
        fd_ctx = m_fdContexts[fd];
    }

    FdContext::MutexType::Lock lock2(fd_ctx->mutex);

    // 添加的事件已经被添加了
    if (SYLAR_UNLIKELY(fd_ctx->events & event)) {
        SYLAR_LOG_ERROR(g_logger)
            << "addEvent assert fd=" << fd << " event=" << (EPOLL_EVENTS)event
            << " fd_ctx.event=" << (EPOLL_EVENTS)fd_ctx->events;
        SYLAR_ASSERT(!(fd_ctx->events & event));
    }

    // 之前有事件？修改事件：添加事件
    int op = fd_ctx->events ? EPOLL_CTL_MOD : EPOLL_CTL_ADD;
    epoll_event epevent;
    epevent.events = EPOLLET | fd_ctx->events | event;
    epevent.data.ptr = fd_ctx; // fdContext通过data.ptr指针保存，在idle中会用到

    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger)
            << "epoll_ctl(" << m_epfd << ", " << (EpollCtlOp)op << ", " << fd
            << ", " << (EPOLL_EVENTS)epevent.events << "):" << rt << " ("
            << errno << ") (" << strerror(errno)
            << ") fd_ctx->events=" << (EPOLL_EVENTS)fd_ctx->events;
        return -1;
    }

    ++m_pendingEventCount;
    fd_ctx->events = (Event)(fd_ctx->events | event);

    // 设置event context
    FdContext::EventContext &event_ctx = fd_ctx->getContext(event);
    SYLAR_ASSERT(!event_ctx.scheduler && !event_ctx.fiber && !event_ctx.cb);

    event_ctx.scheduler = Scheduler::GetThis();
    if (cb) {
        event_ctx.cb.swap(cb);
    } else {
        event_ctx.fiber = Fiber::GetThis(); // 返回到当前协程
        SYLAR_ASSERT2(event_ctx.fiber->getState() == Fiber::EXEC,
                      "state=" << event_ctx.fiber->getState());
    }
    return 0;
}

bool IOManager::delEvent(int fd, Event event) {
    // 找到对应的FdContext
    RWMutexType::ReadLock lock(m_mutex);
    if ((int)m_fdContexts.size() <= fd) {
        return false;
    }
    FdContext *fd_ctx = m_fdContexts[fd];
    lock.unlock();

    FdContext::MutexType::Lock lock2(fd_ctx->mutex);
    if (SYLAR_UNLIKELY(!(fd_ctx->events & event))) {
        return false;
    }

    Event new_events = (Event)(fd_ctx->events & ~event); // 删除event
    int op = new_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events = EPOLLET | new_events;
    epevent.data.ptr = fd_ctx;

    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger)
            << "epoll_ctl(" << m_epfd << ", " << (EpollCtlOp)op << ", " << fd
            << ", " << (EPOLL_EVENTS)epevent.events << "):" << rt << " ("
            << errno << ") (" << strerror(errno) << ")";
        return false;
    }

    --m_pendingEventCount;
    fd_ctx->events = new_events;
    FdContext::EventContext &event_ctx = fd_ctx->getContext(event); 
    fd_ctx->resetContext(event_ctx);// 将event对应的EventContext重置
    return true;
}

// 取消事件会触发该事件
bool IOManager::cancelEvent(int fd, Event event) {
    RWMutexType::ReadLock lock(m_mutex);
    if ((int)m_fdContexts.size() <= fd) {
        return false;
    }
    FdContext *fd_ctx = m_fdContexts[fd];
    lock.unlock();

    FdContext::MutexType::Lock lock2(fd_ctx->mutex);
    if (SYLAR_UNLIKELY(!(fd_ctx->events & event))) {
        return false;
    }

    Event new_events = (Event)(fd_ctx->events & ~event);
    int op = new_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events = EPOLLET | new_events;
    epevent.data.ptr = fd_ctx;

    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger)
            << "epoll_ctl(" << m_epfd << ", " << (EpollCtlOp)op << ", " << fd
            << ", " << (EPOLL_EVENTS)epevent.events << "):" << rt << " ("
            << errno << ") (" << strerror(errno) << ")";
        return false;
    }

    // cancel时，还会触发一次？
    fd_ctx->triggerEvent(event);
    --m_pendingEventCount;
    return true;
}

bool IOManager::cancelAll(int fd) {
    RWMutexType::ReadLock lock(m_mutex);
    if ((int)m_fdContexts.size() <= fd) {
        return false;
    }
    FdContext *fd_ctx = m_fdContexts[fd];
    lock.unlock();

    FdContext::MutexType::Lock lock2(fd_ctx->mutex);
    if (!fd_ctx->events) {
        return false;
    }

    int op = EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events = 0;
    epevent.data.ptr = fd_ctx;

    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger)
            << "epoll_ctl(" << m_epfd << ", " << (EpollCtlOp)op << ", " << fd
            << ", " << (EPOLL_EVENTS)epevent.events << "):" << rt << " ("
            << errno << ") (" << strerror(errno) << ")";
        return false;
    }

    if (fd_ctx->events & READ) {
        fd_ctx->triggerEvent(READ);
        --m_pendingEventCount;
    }
    if (fd_ctx->events & WRITE) {
        fd_ctx->triggerEvent(WRITE);
        --m_pendingEventCount;
    }

    SYLAR_ASSERT(fd_ctx->events == 0);
    return true;
}
```



### idle

IOManager中的idle和tickle方法重写了其父类Scheduler中的相应的方法。在Scheduler中idle函数几乎啥也不干，直接返回调度协程，但是在IOManager中，idle中执行了重要的epoll_wait方法，并在事件发生时触发相应事件；Scheduler中的tickle方法啥也没干，但在IOManager中，由于epoll_wait可能发生阻塞，则添加新的任务时，我们需要在ticket中唤醒处于epoll_wait的协程，让其能够执行新任务，

```c++
// 判断IOManager是否该停止
bool IOManager::stopping(uint64_t &timeout) {
    timeout = getNextTimer();
    // 没有定时器 + 没有待执行的任务 + shceduler调用了stop方法
    return timeout == ~0ull && m_pendingEventCount == 0 &&
           Scheduler::stopping();
}

// 重写stopping
bool IOManager::stopping() {
    uint64_t timeout = 0;
    return stopping(timeout);
}

void IOManager::idle() {
    SYLAR_LOG_DEBUG(g_logger) << "idle";

    // 为epoll event分配空间，epoll_wait返回的事件会填充到events中
    const uint64_t MAX_EVNETS = 256;
    epoll_event *events = new epoll_event[MAX_EVNETS]();
    std::shared_ptr<epoll_event> shared_events(
        events, [](epoll_event *ptr) { delete[] ptr; });

    while (true) {
        uint64_t next_timeout = 0;
        
        //  若stopping，则退出idle协程，注意，会修改next_timeout
        if (SYLAR_UNLIKELY(stopping(next_timeout))) {
            tickle();
            SYLAR_LOG_INFO(g_logger)
                << "name=" << getName() << " idle stopping exit";
            break;
        }

        int rt = 0;
        do {
            static const int MAX_TIMEOUT = 3000;
            if (next_timeout != ~0ull) {
                next_timeout = (int)next_timeout > MAX_TIMEOUT ? MAX_TIMEOUT
                                                               : next_timeout;
            } else {
                next_timeout = MAX_TIMEOUT;
            }
            /*
             epoll_wait会有线程安全问题吗？多个线程都会对同一个epfd调用epoll_wait/epoll_ctl -> 结论：不会
             https://www.zhihu.com/question/49741301
             * 阻塞在这里，但有3中情况能够唤醒epoll_wait
             * 1. 超时时间到了
             * 2. 关注的 socket 有数据来了
             * 3. 通过 tickle 往 pipe 里发数据，表明有任务来了
             */
            rt = epoll_wait(m_epfd, events, MAX_EVNETS, (int)next_timeout);
            if (rt < 0 && errno == EINTR) {
            } else {
                break;
            }
        } while (true);

        std::vector<std::function<void()>> cbs;
        // 获取已经超时的任务，后文介绍，可先略过
        listExpiredCb(cbs);

        // 全部放到任务队列中
        if (!cbs.empty()) {
            // SYLAR_LOG_DEBUG(g_logger) << "on timer cbs.size=" << cbs.size();
            schedule(cbs.begin(), cbs.end());
            cbs.clear();
        }

        // 遍历已经准备好的fd
        for (int i = 0; i < rt; ++i) {
            // 从events中拿一个envent
            epoll_event &event = events[i];

            // 如果获得的这个信息是来自Pipe
            if (event.data.fd == m_tickleFds[0]) {
                uint8_t dummy[256];
                while (read(m_tickleFds[0], dummy, sizeof(dummy)) > 0)
                    ;
                continue; // 跳过来自pipe的event
            }

            // 从ptr中拿出FdContext
            FdContext *fd_ctx = (FdContext *)event.data.ptr;
            FdContext::MutexType::Lock lock(fd_ctx->mutex);
            
            // 如果发生了异常/错误，会触发注册到该fd的所有事件
            if (event.events & (EPOLLERR | EPOLLHUP)) {
                event.events |= (EPOLLIN | EPOLLOUT) & fd_ctx->events;
            }
            int real_events = NONE;
            if (event.events & EPOLLIN) {
                real_events |= READ;
            }
            if (event.events & EPOLLOUT) {
                real_events |= WRITE;
            }

            if ((fd_ctx->events & real_events) == NONE) {
                continue;
            }

            // 剩余的事件
            int left_events = (fd_ctx->events & ~real_events);
            int op = left_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
            event.events = EPOLLET | left_events;

            int rt2 = epoll_ctl(m_epfd, op, fd_ctx->fd, &event);
            if (rt2) {
                SYLAR_LOG_ERROR(g_logger)
                    << "epoll_ctl(" << m_epfd << ", " << (EpollCtlOp)op << ", "
                    << fd_ctx->fd << ", " << (EPOLL_EVENTS)event.events
                    << "):" << rt2 << " (" << errno << ") (" << strerror(errno)
                    << ")";
                continue;
            }

            // 读事件准备好了，执行读事件
            if (real_events & READ) {
                fd_ctx->triggerEvent(READ);
                --m_pendingEventCount;
            }

            // 写事件准备好了，执行写事件
            if (real_events & WRITE) {
                fd_ctx->triggerEvent(WRITE);
                --m_pendingEventCount;
            }
        }
		
        // 回到调度协程
        Fiber::GetThis()->Yield();
    }
}

void IOManager::tickle() {
    // 没有Idle threads，不用唤醒
    if (!hasIdleThreads()) {
        return;
    }

    // 唤醒idle线程
    int rt = write(m_tickleFds[1], "T", 1);
    SYLAR_ASSERT(rt == 1);
}
```



### 定时器概述

sylar的定时器采用优先队列设计，所有定时器根据绝对的超时时间点进⾏排序，每次取出离当前时间最近的⼀个超时时间点，计算出超时需要等待的时间，然后等待超时。超时时间到后，获取当前的绝对时间点，然后把最小堆里超时时间点⼩于这个时间点的定时器都收集起来，执⾏它们的回调函数。

注意，在注册定时事件时，⼀般提供的是相对时间，⽐如相对当前时间3秒后执⾏。sylar会根据传⼊的相对时间和当前的绝对时间计算出定时器超时时的绝对时间点，然后根据这个绝对时间点对定时器进⾏最⼩堆排序。

sylar定时器的超时等待基于epoll_wait，精度只⽀持毫秒级，因为epoll_wait的超时精度也只有毫秒级。

关于定时器和IO协程调度器的整合。IO协程调度器的idle协程会在调度器空闲时阻塞在epoll_wait上，等待IO事件发⽣。在之前的代码⾥，epoll_wait具有固定的超时时间，这个值是5秒钟。加⼊定时器功能后，epoll_wait的超时时间改⽤当前定时器的最⼩超时时间来代替。epoll_wait返回后，根据当前的绝对时间把已超时的所有定时器收集起来，执⾏它们的回调函数。

由于epoll_wait的返回并不⼀定是超时引起的，也有可能是IO事件唤醒的，所以在epoll_wait返回后不能想当然地假设定时器已经超时了，⽽是要再判断⼀下定时器有没有超时，这时绝对时间的好处就体现出来了，通过⽐较当前的绝对时间和定时器的绝对超时时间，就可以确定⼀个定时器到底有没有超时。

所有的timer通过一个set保存下来，set底层是一个红黑树，这是有序的

```c++
std::set<Timer::ptr, Timer::Comparator> m_timers;
```



### Timer

这是对一个定时器的封装，提供取消定时器(cancel)，重置定时器(reset)，刷新定时器(refresh)的方法，含有一个TimerManager的成员，对实行上述操作提供支持

```c++
// 取消定时器
bool Timer::cancel() {
    TimerManager::RWMutexType::WriteLock lock(m_manager->m_mutex);
    if(m_cb) {
        m_cb = nullptr;
        // 在timerManager成员中找到timer，然后删除，仅此而已
        auto it = m_manager->m_timers.find(shared_from_this());
        m_manager->m_timers.erase(it);
        return true;
    }
    return false;
}

// 刷新定时器
// 提供给周期的任务，任务执行完毕后，刷新定时器
bool Timer::refresh() {
    TimerManager::RWMutexType::WriteLock lock(m_manager->m_mutex);
    if(!m_cb) {
        return false;
    }
    auto it = m_manager->m_timers.find(shared_from_this());
    if(it == m_manager->m_timers.end()) {
        return false;
    }
    m_manager->m_timers.erase(it);
    m_next = sylar::GetCurrentMS() + m_ms;
    m_manager->m_timers.insert(shared_from_this());
    return true;
}

// 重置定时器，from_now:是否从当前时间开始计算
bool Timer::reset(uint64_t ms, bool from_now) {
    // 若周期和当前周期相同，且不是从当前时间开始计算
    if(ms == m_ms && !from_now) {
        return true;
    }
    TimerManager::RWMutexType::WriteLock lock(m_manager->m_mutex);
    if(!m_cb) {
        return false;
    }
    auto it = m_manager->m_timers.find(shared_from_this());
    if(it == m_manager->m_timers.end()) {
        return false;
    }
    m_manager->m_timers.erase(it);
    uint64_t start = 0;
    if(from_now) {
        start = sylar::GetCurrentMS();
    } else {
        start = m_next - m_ms; // 反推开始时间
    }
    m_ms = ms;
    m_next = start + m_ms;
    m_manager->addTimer(shared_from_this(), lock);
    return true;

}
```





### TimerManager

TimerManager对Timer进行管理。IOManager继承了TimerManager，因此可以使用TimerManager的所有方法，比如addTimer, addConditionTimer等。

```c++
// 添加定时器，持有lock调用
void TimerManager::addTimer(Timer::ptr val, RWMutexType::WriteLock& lock) {
    auto it = m_timers.insert(val).first;

    // 是最早截止的定时器，并且没有触发过onTimerInsertedAtFront
    bool at_front = (it == m_timers.begin()) && !m_tickled;
    if(at_front) {
        m_tickled = true; // 触发onTimerInsertedAtFront
    }
    lock.unlock();

    if(at_front) {
        onTimerInsertedAtFront(); // 当有定时器插入到首部时回调
    }
}

Timer::ptr TimerManager::addTimer(uint64_t ms, std::function<void()> cb
                                  ,bool recurring) {
    Timer::ptr timer(new Timer(ms, cb, recurring, this));
    RWMutexType::WriteLock lock(m_mutex);
    addTimer(timer, lock);
    return timer;
}

static void OnTimer(std::weak_ptr<void> weak_cond, std::function<void()> cb) {
    // 使用weak_cond的lock函数获取一个shared_ptr指针tmp
    std::shared_ptr<void> tmp = weak_cond.lock();
    // 若tmp不为空，则调用回调函数cb
    if(tmp) {
        cb();
    }
}

// 条件定时器，只有定时器触发时weak_cond仍有效才调用
Timer::ptr TimerManager::addConditionTimer(uint64_t ms, std::function<void()> cb
                                    ,std::weak_ptr<void> weak_cond
                                    ,bool recurring) {
    return addTimer(ms, std::bind(&OnTimer, weak_cond, cb), recurring);
}

/*
return
    有过期的，返回0； 
    没有定时器，返回无穷大
    有定时器，返回最近过期需要的时间
*/ 
uint64_t TimerManager::getNextTimer() {
    RWMutexType::ReadLock lock(m_mutex);
    m_tickled = false;
    if(m_timers.empty()) {
        return ~0ull; // 无穷大
    }

    const Timer::ptr& next = *m_timers.begin();
    uint64_t now_ms = sylar::GetCurrentMS();
    if(now_ms >= next->m_next) {
        return 0;
    } else {
        return next->m_next - now_ms;
    }
}

// 获取所有过期的定时器，并将他们的回调函数填入cbs
void TimerManager::listExpiredCb(std::vector<std::function<void()> >& cbs) {
    uint64_t now_ms = sylar::GetCurrentMS();
    std::vector<Timer::ptr> expired;
    {
        RWMutexType::ReadLock lock(m_mutex);
        if(m_timers.empty()) {
            return;
        }
    }
    RWMutexType::WriteLock lock(m_mutex);
    if(m_timers.empty()) {
        return;
    }
    bool rollover = detectClockRollover(now_ms);
    // 没有时间倒流，并且最早的定时器还没有到期
    if(!rollover && ((*m_timers.begin())->m_next > now_ms)) {
        return;
    }

    // 找到所有过期的定时器
    Timer::ptr now_timer(new Timer(now_ms));
    auto it = rollover ? m_timers.end() : m_timers.lower_bound(now_timer);
    while(it != m_timers.end() && (*it)->m_next == now_ms) {
        ++it;
    }
    expired.insert(expired.begin(), m_timers.begin(), it);
    m_timers.erase(m_timers.begin(), it);
    cbs.reserve(expired.size());

    for(auto& timer : expired) {
        cbs.push_back(timer->m_cb);
        if(timer->m_recurring) { // 如果循环，则重新添加
            timer->m_next = now_ms + timer->m_ms;
            m_timers.insert(timer);
        } else {
            timer->m_cb = nullptr;
        }
    }
}

bool TimerManager::detectClockRollover(uint64_t now_ms) {
    bool rollover = false;
    // 如果当前时间比上次执行时间还小 并且 小于一个小时的时间，相当于时间倒流了
    if(now_ms < m_previouseTime &&
            now_ms < (m_previouseTime - 60 * 60 * 1000)) {
        rollover = true;
    }
    m_previouseTime = now_ms; // 重新调整时间
    return rollover;
}

bool TimerManager::hasTimer() {
    RWMutexType::ReadLock lock(m_mutex);
    return !m_timers.empty();
}

// IOManager实现了onTimerInsertAtFront方法
void IOManager::onTimerInsertedAtFront() { tickle(); }
```


# sylar学习09：IO协程调度模块

这个模块感觉有点难，所以慢一点看好了。

**I/O模式**

应用程序发起一次IO访问分为两个阶段:

1. IO调用阶段：应用程序向内核发起系统调用。

2. IO执行阶段：内核执行IO操作并返回。

   数据准备阶段：内核等待IO设备准备好数据

   数据拷贝阶段：将数据从内核缓冲区拷贝到用户空间缓冲区

**阻塞I/O**

在默认情况下，所有的`socket`都是被阻塞的，也就是**阻塞I/O**，这样就会导致**两个阶段的阻塞**：等待数据 、从内核拷贝数据到用户空间 。

**非阻塞I/O**

可以通过设置`socket`变为`non-blocking`，如果内核还没有准备好数据，那么并不会阻塞用户进程，而是立刻返回一个`error`，可以通过系统调用获得数据，一旦内核的数据准备好了，并且又再次收到了用户进程的`system call`，那么它马上就将数据拷贝到了用户内存（**第二阶段的阻塞**：在拷贝的过程中进程也是会被block），然后返回。

相当于是第一个阶段阻塞，第二个阶段不阻塞。

**异步IO**

在两个阶段都不会被阻塞。**第一阶段：** 例如，当用户进程发起`read`操作后，内核收到用户进程的`system call`会立刻返回，不会对用户进程产生任何的阻塞，**第二阶段：** 当内核准备好了数据，将数据拷贝到用户空间，当这一切都完成之后，内核才会给用户进程发送信号表示操作完成，所以第二阶段也不会被阻塞。

两个阶段都没有阻塞，其实有点神奇，不清楚第二个阶段是怎么做到可以不阻塞的。

**I/O多路复用**

服务器要跟多个客户端建立连接，就需要处理大量的`socket fd`，通过单线程或单进程同时监测若干个文件描述符是否可以执行IO操作，这就是IO多路复用。

当用户进程调用了`select`，那么整个进程都会被阻塞，同时，内核会监视所有`select`负责的`socket fd`，当任何一个`socket`中的数据准备好了，`select`就会返回，这个时候用户进程在调用`recvfrom`操作，内核收到`system call`就将数据拷贝到用户进程。

select相当于一个多路监视器，负责多个用户，当对应的数据准备好之后通知他，然后再进行交互的信息传递。

关于epoll其实还有很多内容，不过后面再看好了，先看一个这个文件。

类IOManager

把socket的事件归类为读事件和写事件。

```C++
/**
     * @brief IO事件，继承自epoll对事件的定义
     * @details 这里只关心socket fd的读和写事件，其他epoll事件会归类到这两类事件中
     */
    enum Event {
        /// 无事件
        NONE = 0x0,
        /// 读事件(EPOLLIN)
        READ = 0x1,
        /// 写事件(EPOLLOUT)
        WRITE = 0x4,
    };
```

IOManager封装了`socket事件上下文`结构体。

```C++
struct FdContext {
        typedef Mutex MutexType;
        /**
         * @brief 事件上下文类
         * @details fd的每个事件都有一个事件上下文，保存这个事件的回调函数以及执行回调函数的调度器
         *          sylar对fd事件做了简化，只预留了读事件和写事件，所有的事件都被归类到这两类事件中
         */
        struct EventContext {
            /// 执行事件回调的调度器
            Scheduler *scheduler = nullptr;
            /// 事件回调协程
            Fiber::ptr fiber;
            /// 事件回调函数
            std::function<void()> cb;
        };

        /**
         * @brief 获取事件上下文类
         * @param[in] event 事件类型
         * @return 返回对应事件的上下文
         */
        EventContext &getEventContext(Event event);

        /**
         * @brief 重置事件上下文
         * @param[in, out] ctx 待重置的事件上下文对象
         */
        void resetEventContext(EventContext &ctx);

        /**
         * @brief 触发事件
         * @details 根据事件类型调用对应上下文结构中的调度器去调度回调协程或回调函数
         * @param[in] event 事件类型
         */
        void triggerEvent(Event event);

        /// 读事件上下文
        EventContext read;
        /// 写事件上下文
        EventContext write;
        /// 事件关联的句柄
        int fd = 0;
        /// 该fd添加了哪些事件的回调函数，或者说该fd关心哪些事件
        Event events = NONE;
        /// 事件的Mutex
        MutexType mutex;
    };
```

获取事件上下文类：

```c++
IOManager::FdContext::EventContext &IOManager::FdContext::getEventContext(IOManager::Event event) {
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
```

重置事件上下文：

```c++
void IOManager::FdContext::resetEventContext(EventContext &ctx) {
    ctx.scheduler = nullptr;
    ctx.fiber.reset();
    ctx.cb = nullptr;
}
```

触发事件：

```c++
void IOManager::FdContext::triggerEvent(IOManager::Event event) {
    // 待触发的事件必须已被注册过
    SYLAR_ASSERT(events & event);
    /**
     *  清除该事件，表示不再关注该事件了
     * 也就是说，注册的IO事件是一次性的，如果想持续关注某个socket fd的读写事件，那么每次触发事件之后都要重新添加
     */
    events = (Event)(events & ~event);
    // 调度对应的协程
    EventContext &ctx = getEventContext(event);
    if (ctx.cb) {
        ctx.scheduler->schedule(ctx.cb);
    } else {
        ctx.scheduler->schedule(ctx.fiber);
    }
    resetEventContext(ctx);
    return;
}
```

注册的IO事件是一次性的，这个和上面说的是一致的。

成员变量：

```c++
/// epoll 文件句柄
int m_epfd = 0;
/// pipe 文件句柄，fd[0]读端，fd[1]写端
int m_tickleFds[2];
/// 当前等待执行的IO事件数量
std::atomic<size_t> m_pendingEventCount = {0};
/// IOManager的Mutex
RWMutexType m_mutex;
/// socket事件上下文的容器
std::vector<FdContext *> m_fdContexts;
```

构造函数：

```c++
IOManager::IOManager(size_t threads, bool use_caller, const std::string &name)
    : Scheduler(threads, use_caller, name) {
    m_epfd = epoll_create(5000); // 创建一个epollfd
    SYLAR_ASSERT(m_epfd > 0);// 成功时，这些系统调用将返回非负文件描述符。如果出错，则返回-1，并且将errno设置为指示错误。

    int rt = pipe(m_tickleFds);// 创建管道，用于进程间通信
    SYLAR_ASSERT(!rt);// 成功返回0，失败返回-1，并且设置errno。

    // 关注pipe读句柄的可读事件，用于tickle协程
    epoll_event event;
    memset(&event, 0, sizeof(epoll_event));// 用0初始化event
    event.events  = EPOLLIN | EPOLLET;// 注册读事件，设置边缘触发模式
    event.data.fd = m_tickleFds[0];// fd关联pipe的读端
    // 对一个打开的文件描述符执行一系列控制操作
    // 非阻塞方式，配合边缘触发，F_SETFL: 获取/设置文件状态标志，O_NONBLOCK: 使I/O变成非阻塞模式，在读取不到数据或是写入缓冲区已满会马上return，而不会阻塞等待。
    rt = fcntl(m_tickleFds[0], F_SETFL, O_NONBLOCK);
    SYLAR_ASSERT(!rt);
    // 将pipe的读端注册到epoll
    rt = epoll_ctl(m_epfd, EPOLL_CTL_ADD, m_tickleFds[0], &event);
    SYLAR_ASSERT(!rt);
    // 初始化socket事件上下文vector
    contextResize(32);

    start();
}
//重置socket句柄上下文的容器大小
void IOManager::contextResize(size_t size) {
    m_fdContexts.resize(size);

    for (size_t i = 0; i < m_fdContexts.size(); ++i) {
        if (!m_fdContexts[i]) {
            m_fdContexts[i]     = new FdContext;
            m_fdContexts[i]->fd = i;
        }
    }
}
```

析构函数：

```c++
IOManager::~IOManager() {
    stop();// 停止调度器
    close(m_epfd);// 释放epoll
    close(m_tickleFds[0]);// 释放pipe
    close(m_tickleFds[1]);

    for (size_t i = 0; i < m_fdContexts.size(); ++i) {// 释放 m_fdContexts 内存
        if (m_fdContexts[i]) {
            delete m_fdContexts[i];
        }
    }
}
```

这个函数就很平铺直叙，可以看懂了。

addEvent（添加事件）

```c++
int IOManager::addEvent(int fd, Event event, std::function<void()> cb) {
    // 找到fd对应的FdContext，如果不存在，那就分配一个
    FdContext *fd_ctx = nullptr;
    RWMutexType::ReadLock lock(m_mutex);
    if ((int)m_fdContexts.size() > fd) {
        fd_ctx = m_fdContexts[fd];
        lock.unlock();
    } else {
        lock.unlock();
        RWMutexType::WriteLock lock2(m_mutex);
        contextResize(fd * 1.5);// 不够就扩充
        fd_ctx = m_fdContexts[fd];
    }

    // 同一个fd不允许重复添加相同的事件， 可能是两个不同的线程在操控同一个句柄添加事件
    FdContext::MutexType::Lock lock2(fd_ctx->mutex);
    if (SYLAR_UNLIKELY(fd_ctx->events & event)) {
        SYLAR_LOG_ERROR(g_logger) << "addEvent assert fd=" << fd
                                  << " event=" << (EPOLL_EVENTS)event
                                  << " fd_ctx.event=" << (EPOLL_EVENTS)fd_ctx->events;
        SYLAR_ASSERT(!(fd_ctx->events & event));
    }
    // 若已经有注册的事件则为修改操作，若没有则为添加操作
    // 将新的事件加入epoll_wait，使用epoll_event的私有指针存储FdContext的位置
    int op = fd_ctx->events ? EPOLL_CTL_MOD : EPOLL_CTL_ADD;
    epoll_event epevent;
    epevent.events   = EPOLLET | fd_ctx->events | event;// 设置边缘触发模式，添加原有的事件以及要注册的事件
    epevent.data.ptr = fd_ctx;// 将fd_ctx存到data的指针中
    // 注册事件
    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
                                  << (EpollCtlOp)op << ", " << fd << ", " << (EPOLL_EVENTS)epevent.events << "):"
                                  << rt << " (" << errno << ") (" << strerror(errno) << ") fd_ctx->events="
                                  << (EPOLL_EVENTS)fd_ctx->events;
        return -1;
    }

    // 待执行IO事件数加1
    ++m_pendingEventCount;

    // 找到这个fd的event事件对应的EventContext，对其中的scheduler, cb, fiber进行赋值
    fd_ctx->events                     = (Event)(fd_ctx->events | event);
    FdContext::EventContext &event_ctx = fd_ctx->getEventContext(event);
    SYLAR_ASSERT(!event_ctx.scheduler && !event_ctx.fiber && !event_ctx.cb);

    // 赋值scheduler和回调函数，如果回调函数为空，则把当前协程当成回调执行体
    event_ctx.scheduler = Scheduler::GetThis();
    if (cb) {
        event_ctx.cb.swap(cb);
    } else {
        event_ctx.fiber = Fiber::GetThis();
        SYLAR_ASSERT2(event_ctx.fiber->getState() == Fiber::RUNNING, "state=" << event_ctx.fiber->getState());
    }
    return 0;
}

bool IOManager::delEvent(int fd, Event event) {
    // 找到fd对应的FdContext
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

    // 清除指定的事件，表示不关心这个事件了，如果清除之后结果为0，则从epoll_wait中删除该文件描述符
    Event new_events = (Event)(fd_ctx->events & ~event);
    int op           = new_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events   = EPOLLET | new_events;
    epevent.data.ptr = fd_ctx;

    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
                                  << (EpollCtlOp)op << ", " << fd << ", " << (EPOLL_EVENTS)epevent.events << "):"
                                  << rt << " (" << errno << ") (" << strerror(errno) << ")";
        return false;
    }

    // 待执行事件数减1
    --m_pendingEventCount;
    // 重置该fd对应的event事件上下文
    fd_ctx->events                     = new_events;
    FdContext::EventContext &event_ctx = fd_ctx->getEventContext(event);
    fd_ctx->resetEventContext(event_ctx);
    return true;
}

bool IOManager::cancelEvent(int fd, Event event) {
    // 找到fd对应的FdContext
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

    // 删除事件
    Event new_events = (Event)(fd_ctx->events & ~event);
    int op           = new_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events   = EPOLLET | new_events;
    epevent.data.ptr = fd_ctx;

    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
                                  << (EpollCtlOp)op << ", " << fd << ", " << (EPOLL_EVENTS)epevent.events << "):"
                                  << rt << " (" << errno << ") (" << strerror(errno) << ")";
        return false;
    }

    // 删除之前触发一次事件
    fd_ctx->triggerEvent(event);
    // 活跃事件数减1
    --m_pendingEventCount;
    return true;
}
```

delEvent（删除事件）

```C++
bool IOManager::cancelAll(int fd) {
    // 找到fd对应的FdContext
    RWMutexType::ReadLock lock(m_mutex);
    if ((int)m_fdContexts.size() <= fd) {
        return false;
    }
    FdContext *fd_ctx = m_fdContexts[fd];// 拿到 fd 对应的 FdContext
    lock.unlock();

    FdContext::MutexType::Lock lock2(fd_ctx->mutex);
    if (!fd_ctx->events) {// 若没有要删除的事件
        return false;
    }

    // 删除全部事件
    int op = EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events   = 0;
    epevent.data.ptr = fd_ctx;// ptr 关联 fd_ctx
    // 注册事件
    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
                                  << (EpollCtlOp)op << ", " << fd << ", " << (EPOLL_EVENTS)epevent.events << "):"
                                  << rt << " (" << errno << ") (" << strerror(errno) << ")";
        return false;
    }

    // 触发全部已注册的事件
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

cancelEvent(取消事件)

取消事件会出发该事件。

```c++
bool IOManager::cancelEvent(int fd, Event event) {
    // 找到fd对应的FdContext
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

    // 删除事件
    Event new_events = (Event)(fd_ctx->events & ~event);
    int op           = new_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events   = EPOLLET | new_events;
    epevent.data.ptr = fd_ctx;

    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
                                  << (EpollCtlOp)op << ", " << fd << ", " << (EPOLL_EVENTS)epevent.events << "):"
                                  << rt << " (" << errno << ") (" << strerror(errno) << ")";
        return false;
    }

    // 删除之前触发一次事件
    fd_ctx->triggerEvent(event);
    // 活跃事件数减1
    --m_pendingEventCount;
    return true;
}
```

cancelAll（取消所有事件）

取消事件会触发该事件

```c++
bool IOManager::cancelAll(int fd) {
    // 找到fd对应的FdContext
    RWMutexType::ReadLock lock(m_mutex);
    if ((int)m_fdContexts.size() <= fd) {
        return false;
    }
    FdContext *fd_ctx = m_fdContexts[fd];// 拿到 fd 对应的 FdContext
    lock.unlock();

    FdContext::MutexType::Lock lock2(fd_ctx->mutex);
    if (!fd_ctx->events) {// 若没有要删除的事件
        return false;
    }

    // 删除全部事件
    int op = EPOLL_CTL_DEL;
    epoll_event epevent;
    epevent.events   = 0;// 没有事件
    epevent.data.ptr = fd_ctx;// ptr 关联 fd_ctx
    // 注册事件
    int rt = epoll_ctl(m_epfd, op, fd, &epevent);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
                                  << (EpollCtlOp)op << ", " << fd << ", " << (EPOLL_EVENTS)epevent.events << "):"
                                  << rt << " (" << errno << ") (" << strerror(errno) << ")";
        return false;
    }

    // 触发全部已注册的事件
    if (fd_ctx->events & READ) {// 有读事件执行读事件
        fd_ctx->triggerEvent(READ);
        --m_pendingEventCount;
    }
    if (fd_ctx->events & WRITE) {// 有写事件执行写事件
        fd_ctx->triggerEvent(WRITE);
        --m_pendingEventCount;
    }

    SYLAR_ASSERT(fd_ctx->events == 0);
    return true;
}
```

上面两个取消事件还是自己不是很明白，感觉差别不是很大mmm。

triggerEvent(触发事件)

根据事件类型调用对应上下文结构中的调度器去调度回调协程或回调函数。

```c++
void IOManager::FdContext::triggerEvent(IOManager::Event event) {
    // 待触发的事件必须已被注册过
    SYLAR_ASSERT(events & event);
    /**
     *  清除该事件，表示不再关注该事件了
     * 也就是说，注册的IO事件是一次性的，如果想持续关注某个socket fd的读写事件，那么每次触发事件之后都要重新添加
     */
    events = (Event)(events & ~event);
    // 调度对应的协程
    EventContext &ctx = getEventContext(event);
    if (ctx.cb) {
        ctx.scheduler->schedule(ctx.cb);
    } else {
        ctx.scheduler->schedule(ctx.fiber);
    }
    resetEventContext(ctx);
    return;
}
```

GetThis(获得当前IO调度器)

```c++
IOManager *IOManager::GetThis() {
    return dynamic_cast<IOManager *>(Scheduler::GetThis());
}
```

tickle(通知调度器有任务要调度)

写pipe让idle协程从epoll_wait退出，待idle协程yield之后Scheduler::run就可以调度其他任务。

```c++
void IOManager::tickle() {
    SYLAR_LOG_DEBUG(g_logger) << "tickle";
    if(!hasIdleThreads()) {//需要有空闲线程才可以继续进行
        return;
    }
    int rt = write(m_tickleFds[1], "T", 1);
    SYLAR_ASSERT(rt == 1);
}
```

stopping（停止条件）

```c++
bool IOManager::stopping() {
    uint64_t timeout = 0;
    return stopping(timeout);
}
bool IOManager::stopping(uint64_t &timeout) {
    // 对于IOManager而言，必须等所有待调度的IO事件都执行完了才可以退出
    // 增加定时器功能后，还应该保证没有剩余的定时器待触发
    timeout = getNextTimer();
    // 定时器为空 && 等待执行的事件数量为0 && scheduler可以stop
    return timeout == ~0ull && m_pendingEventCount == 0 && Scheduler::stopping();
}
```

idle(idle协程)

对于IO协程调度来说，应阻塞在等待IO事件上，idle退出的时机是epoll_wait返回，对应的操作是tickle或注册的IO事件发生。

```c++
void IOManager::idle() {
    SYLAR_LOG_DEBUG(g_logger) << "idle";

    // 一次epoll_wait最多检测256个就绪事件，如果就绪事件超过了这个数，那么会在下轮epoll_wati继续处理
    const uint64_t MAX_EVNETS = 256;
    epoll_event *events       = new epoll_event[MAX_EVNETS]();
    std::shared_ptr<epoll_event> shared_events(events, [](epoll_event *ptr) {// 使用智能指针托管events， 离开idle自动释放
        delete[] ptr;
    });

    while (true) {
        // 获取下一个定时器的超时时间，顺便判断调度器是否停止
        uint64_t next_timeout = 0;
        if( SYLAR_UNLIKELY(stopping(next_timeout))) {
            SYLAR_LOG_DEBUG(g_logger) << "name=" << getName() << "idle stopping exit";
            break;
        }

        // 阻塞在epoll_wait上，等待事件发生或定时器超时
        int rt = 0;
        do{
            // 默认超时时间5秒，如果下一个定时器的超时时间大于5秒，仍以5秒来计算超时，避免定时器超时时间太大时，epoll_wait一直阻塞
            static const int MAX_TIMEOUT = 5000;
            if(next_timeout != ~0ull) {
                next_timeout = std::min((int)next_timeout, MAX_TIMEOUT);
            } else {
                next_timeout = MAX_TIMEOUT;
            }//阻塞在这里
            rt = epoll_wait(m_epfd, events, MAX_EVNETS, (int)next_timeout);
            if(rt < 0 && errno == EINTR) {//这里就是源码 ep_poll() 中由操作系统中断返回的 EINTR，需要重新尝试 epoll_Wait
                continue;
            } else {
                break;
            }
        } while(true);

        // 收集所有已超时的定时器，执行回调函数
        std::vector<std::function<void()>> cbs;
        listExpiredCb(cbs);
        if(!cbs.empty()) {
            for(const auto &cb : cbs) {
                schedule(cb);// 全部放到任务队列中
            }
            cbs.clear();
        }
        
        // 遍历所有发生的事件，根据epoll_event的私有指针找到对应的FdContext，进行事件处理
        for (int i = 0; i < rt; ++i) {
            epoll_event &event = events[i];
            if (event.data.fd == m_tickleFds[0]) {// 如果获得的这个信息时来自 pipe
                // ticklefd[0]用于通知协程调度，这时只需要把管道里的内容读完即可
                uint8_t dummy[256];
                while (read(m_tickleFds[0], dummy, sizeof(dummy)) > 0)// 将 pipe 发来的1个字节数据读掉
                    ;
                continue;
            }

            FdContext *fd_ctx = (FdContext *)event.data.ptr;
            FdContext::MutexType::Lock lock(fd_ctx->mutex);
            /**
             * EPOLLERR: 出错，比如写读端已经关闭的pipe
             * EPOLLHUP: 套接字对端关闭
             * 出现这两种事件，应该同时触发fd的读和写事件，否则有可能出现注册的事件永远执行不到的情况
             */ 
            if (event.events & (EPOLLERR | EPOLLHUP)) {
                event.events |= (EPOLLIN | EPOLLOUT) & fd_ctx->events;
            }
            int real_events = NONE;
            if (event.events & EPOLLIN) {// 读事件好了
                real_events |= READ;
            }
            if (event.events & EPOLLOUT) {// 写事件好了
                real_events |= WRITE;
            }

            if ((fd_ctx->events & real_events) == NONE) {// 没事件
                continue;
            }

            // 剔除已经发生的事件，将剩下的事件重新加入epoll_wait
            int left_events = (fd_ctx->events & ~real_events);
            int op          = left_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;// 如果执行完该事件还有事件则修改，若无事件则删除
            event.events    = EPOLLET | left_events;// 更新新的事件

            int rt2 = epoll_ctl(m_epfd, op, fd_ctx->fd, &event);// 重新注册事件
            if (rt2) {
                SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
                                          << (EpollCtlOp)op << ", " << fd_ctx->fd << ", " << (EPOLL_EVENTS)event.events << "):"
                                          << rt2 << " (" << errno << ") (" << strerror(errno) << ")";
                continue;
            }

            // 处理已经发生的事件，也就是让调度器调度指定的函数或协程
            if (real_events & READ) {// 读事件好了，执行读事件
                fd_ctx->triggerEvent(READ);
                --m_pendingEventCount;
            }
            if (real_events & WRITE) {// 写事件好了，执行写事件
                fd_ctx->triggerEvent(WRITE);
                --m_pendingEventCount;
            }
        } // end for

        /**
         * 一旦处理完所有的事件，idle协程yield，这样可以让调度协程(Scheduler::run)重新检查是否有新任务要调度
         * 上面triggerEvent实际也只是把对应的fiber重新加入调度，要执行的话还要等idle协程退出
         */ 
        Fiber::ptr cur = Fiber::GetThis();// 获得当前协程
        auto raw_ptr   = cur.get();// 获得裸指针
        cur.reset();

        raw_ptr->yield();// 执行完返回scheduler的MainFiber 继续下一轮
    } // end while(true)
}
```





关于epoll

是之前`select`和`poll`的增强版本，更下灵活，没有描述符的限制。`epoll`使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的`copy`只需一次。

包含三个接口函数：

函数一：epoll_create

```c++
//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_create(int size)；
```

当创建好`epoll`句柄后，它就会占用一个`fd`值，在linux下如果查看`/proc/进程id/fd/`，是能够看到这个`fd`的，所以在使用完`epoll`后，必须调用`close()`关闭，否则可能导致`fd`被耗尽。

> 我们知道在Linux系统中一切皆可以看成是文件，文件又可分为：普通文件、目录文件、链接文件和设备文件。在操作这些所谓的文件的时候，我们每操作一次就找一次名字，这会耗费大量的时间和效率。所以Linux中规定每一个文件对应一个索引，这样要操作文件的时候，我们直接找到索引就可以对其进行操作了。
>
> 文件描述符（file descriptor）就是内核为了高效管理这些已经被打开的文件所创建的索引，其是一个非负整数（通常是小整数），用于指代被打开的文件，所有执行I/O操作的系统调用都通过文件描述符来实现。同时还规定系统刚刚启动的时候，0是标准输入，1是标准输出，2是标准错误。这意味着如果此时去打开一个新的文件，它的文件描述符会是3，再打开一个文件文件描述符就是4......
>
> Linux内核对所有打开的文件有一个文件描述符表格，里面存储了每个文件描述符作为索引与一个打开文件相对应的关系，简单理解就是下图这样一个数组，文件描述符（索引）就是文件描述符表这个数组的下标，数组的内容就是指向一个个打开的文件的指针。

每创建一个`epollfd`, 内核就会分配一个`eventpoll`结构体与之对应，其中维护了一个`RBTree`来存放所有要监听的`struct epitem(表示一个被监听的fd)`

函数二：epoll_ctl

从用户空间将`epoll_event`结构copy到内核空间。

```C++
/*
    @param epfd epoll_create()的返回值
    @param op 添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD事件
    @param event 告诉内核需要监听什么事
*/
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
    
struct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};

/* 
 * epoll事件关联数据的联合体
 * fd： 表示关联的文件描述符。
 * ptr：表示关联的指针。
 */
typedef union epoll_data {
    void *ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;

//events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
```

同一个`fd`不能重复添加。内核会自动添加这两个事件`epds.events |= POLLERR | POLLHUP;`(感觉是先添加意外情况，然后对应什么情况就添加什么宏)并且使用`copy_from_user`从用户空间将`epoll_event`结构copy到内核空间。

函数三：epoll_wait

```C++
/*
    @param epfd epoll_create() 返回的句柄
    @param events 分配好的 epoll_event 结构体数组，epoll 将会把发生的事件复制到 events 数组中
                  events不可以是空指针，内核只负责把数据复制到这个 events 数组中，不会去帮助我们在用户态中分配内存，但是内核会检查空间是否合法
    @param maxevents 表示本次可以返回的最大事件数目，通常 maxevents 参数与预分配的 events 数组的大小是相等的；
    @param timeout 表示在没有检测到事件发生时最多等待的时间（单位为毫秒）
                   如果 timeout 为 0，则表示 epoll_wait 在 rdllist 链表为空时，立刻返回，不会等待。
                   rdllist:所有已经ready的epitem(表示一个被监听的fd)都在这个链表里面
                   
*/
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

收集在 `epoll`监控的事件中已经发生的事件，如果`epoll`中没有任何一个事件发生，则最多等待`timeout`毫秒后返回。`epoll_wait`的返回值表示当前发生的事件个数，如果返回 0，则表示本次调用中没有事件发生，如果返回 -1，则表示发生错误，需要检查`errno`判断错误类型。



这里的主函数异常的简洁：

```c++
int main(int argc, char *argv[]) {
    sylar::EnvMgr::GetInstance()->init(argc, argv);
    sylar::Config::LoadFromConfDir(sylar::EnvMgr::GetInstance()->getConfigPath());
    
    test_iomanager();

    return 0;
}
```

除了处理命令行，读入配置文件，就是对类IOManager的处理，即test_iomanager()函数。

```c++
void test_iomanager() {
    sylar::IOManager iom;
    // sylar::IOManager iom(10); // 演示多线程下IO协程在不同线程之间切换
    iom.schedule(test_io);
}
```

首先是对IOManager的初始化：

```c++
IOManager::IOManager(size_t threads, bool use_caller, const std::string &name)
    : Scheduler(threads, use_caller, name) {
    m_epfd = epoll_create(5000); // 创建一个epollfd
    SYLAR_ASSERT(m_epfd > 0);// 成功时，这些系统调用将返回非负文件描述符。如果出错，则返回-1，并且将errno设置为指示错误。

    int rt = pipe(m_tickleFds);// 创建管道，用于进程间通信
    SYLAR_ASSERT(!rt);// 成功返回0，失败返回-1，并且设置errno。

    // 关注pipe读句柄的可读事件，用于tickle协程
    epoll_event event;
    memset(&event, 0, sizeof(epoll_event));// 用0初始化event
    event.events  = EPOLLIN | EPOLLET;// 注册读事件，设置边缘触发模式
    event.data.fd = m_tickleFds[0];// fd关联pipe的读端
    // 对一个打开的文件描述符执行一系列控制操作
    // 非阻塞方式，配合边缘触发，F_SETFL: 获取/设置文件状态标志，O_NONBLOCK: 使I/O变成非阻塞模式，在读取不到数据或是写入缓冲区已满会马上return，而不会阻塞等待。
    rt = fcntl(m_tickleFds[0], F_SETFL, O_NONBLOCK);
    SYLAR_ASSERT(!rt);
    // 将pipe的读端注册到epoll
    rt = epoll_ctl(m_epfd, EPOLL_CTL_ADD, m_tickleFds[0], &event);
    SYLAR_ASSERT(!rt);
    // 初始化socket事件上下文vector
    contextResize(32);

    start();
}
```

首先类IOManager继承了类Scheduler，所以首先要初始化一下类scheduler，然后再初始化IOManager。初始化类scheduler这里就不介绍了。

这里创建了两个东西，一个是m_epfd，这个是epollfd，m_epfd这个值实际上为句柄。就像上面学习的，即是打开文件，是从3开始标号，查看发现，这里的句柄值恰好为3。

![image-20240702174657366](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240702174657366.png)

然后是创建了管道。就像之前学习的，管道有一个读端，有一个写端。返回值为0，表示成功建立管道。同样的m_tickleFds中也是两位句柄，查看后发现分别为4和5。

![image-20240702175905379](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240702175905379.png)

之后建立了一个epoll_event结构的变量event。

```c++
typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;	/* Epoll events */
  epoll_data_t data;	/* User data variable */
} __EPOLL_PACKED;
```

进行注册：

```c++
event.events  = EPOLLIN | EPOLLET;// 注册读事件，设置边缘触发模式
event.data.fd = m_tickleFds[0];// fd关联pipe的读端
```

可以看到event用于处理读事件。

关于边缘触发模式，与之对应的是水平触发模式。

> 在水平触发模式下，应用程序需要持续处理就绪的文件描述符，直到文件描述符上不再有数据可读为止。 相比之下，在边缘触发模式下，应用程序只会在状态变化时收到通知，而不会持续收到通知直到处理完数据。

简而言之，边缘触发模式关注的一瞬间的变化，水平触发模式持续关注。

另外关于events的值，其实相当于每一位表示一个注册值，如果这个值为1，说明对应的这一位注册了。

![image-20240702175958436](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240702175958436.png)

关联之后的events值，其中2147483649为2^31+1，即对应第0位和第31位。data中fd值变为4。不过data里面其他值也变为了4，感觉是同步变化的这样。

之后就是将pipe的读端注册到epoll上。

目前来看其实就三个结构玩来玩去，m_epfd，m_tickleFds，event。

之后就是schedule操作了。

```c++
iom.schedule(test_io);

template <class FiberOrCb>
    void schedule(FiberOrCb fc, int thread = -1) {
        bool need_tickle = false;
        {
            MutexType::Lock lock(m_mutex);
            need_tickle = scheduleNoLock(fc, thread);
        }

        if (need_tickle) {
            tickle(); // 唤醒idle协程（进行输出打印）
        }
    }
```

这个其实也没有操作什么，就是添加了一个task任务。

所以其实主要是析构函数的操作hh。

```c++
IOManager::~IOManager() {
    stop();// 停止调度器
    close(m_epfd);// 释放epoll
    close(m_tickleFds[0]);// 释放pipe
    close(m_tickleFds[1]);

    for (size_t i = 0; i < m_fdContexts.size(); ++i) {// 释放 m_fdContexts 内存
        if (m_fdContexts[i]) {
            delete m_fdContexts[i];
        }
    }
}
```

首先是stop函数停止调度器，这个之前见过hh。主体就是进行resume=>MainFunc=>run。

这里就不展开说了。

最后落到函数`test_io`。
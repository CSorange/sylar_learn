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
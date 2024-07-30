# sylar22：再探scheduler/iomanager

可以看到，test_iomanager.cc中其实主要就是函数test_iomanager。

```c++
void test_iomanager() {
    sylar::IOManager iom;
    // sylar::IOManager iom(10); // 演示多线程下IO协程在不同线程之间切换
    iom.schedule(test_io);
}
```

首先就是iomanager的初始化。

```c++
IOManager::IOManager(size_t threads, bool use_caller, const std::string &name)
    : Scheduler(threads, use_caller, name) {...}
```

可以看到，IOManager是基于Scheduler的，所以我们需要首先初始化Scheduler。

```c++
Scheduler::Scheduler(size_t threads, bool use_caller, const std::string &name) {
    SYLAR_ASSERT(threads > 0);// 确定线程数量要正确

    m_useCaller = use_caller; 
    m_name      = name;

    if (use_caller) {// 是否将协程调度线程也纳入调度器
        --threads;
        sylar::Fiber::GetThis();// 获得主协程,这里没有，所以要重新生成。
        SYLAR_ASSERT(GetThis() == nullptr);
        t_scheduler = this; // 设置当前协程调度器

        /**
         * caller线程的主协程不会被线程的调度协程run进行调度，而且，线程的调度协程停止时，应该返回caller线程的主协程
         * 在user caller情况下，把caller线程的主协程暂时保存起来，等调度协程结束时，再resume caller协程
         */
        m_rootFiber.reset(new Fiber(std::bind(&Scheduler::run, this), 0, false));
        sylar::Thread::SetName(m_name);
        //m_rootFiber为子协程，这里的m_rootFiber是调度协程（执行run任务的协程），只有默认构造出来的fiber才是主协程
        t_scheduler_fiber = m_rootFiber.get();
        m_rootThread      = sylar::GetThreadId();// 获得当前线程id
        m_threadIds.push_back(m_rootThread);
    } else {// 不将当前线程纳入调度器
        m_rootThread = -1;
    }
    m_threadCount = threads;
}
```

这里其实主要就做了一件事情，除了初始化成员变量以外，就是判断是否将协程调度线程也纳入调度器。其实也可以理解，毕竟这个Scheduler结构就是用来进行调度的。

如果将协程调度线程放入调度器，后面会出现很多为0的，比如m_threadCount，然后很多循环就走不了了，感觉还听奇怪的。

如果放入，这里就需要构造调度协程。当然在构造子协程之前还需要构造主协程。是的这里的m_rootFiber其实是子协程，但也是调度子协程，这个在之后resume进行切换可以看到，进行上下文交换时，如果在调度，就是交换调度协程，而非主协程。



emmm发现shedule也有对应的，所以还是先看看test_scheduler.cc吧。

```c++
// 添加调度任务，使用函数作为调度对象
    sc.schedule(test_fiber1);
    sc.schedule(test_fiber2);

    // 添加调度任务，使用Fiber类作为调度对象
    sylar::Fiber::ptr fiber(new sylar::Fiber(&test_fiber3));
    sc.schedule(fiber);

    // 创建调度线程，开始任务调度，如果只使用main函数线程进行调度，那start相当于什么也没做
    sc.start();

    /**
     * 只要调度器未停止，就可以添加调度任务
     * 包括在子协程中也可以通过sylar::Scheduler::GetThis()->scheduler()的方式继续添加调度任务
     */
    sc.schedule(test_fiber4);

    /**
     * 停止调度，如果未使用当前线程进行调度，那么只需要简单地等所有调度线程退出即可
     * 如果使用了当前线程进行调度，那么要先执行当前线程的协程调度函数，等其执行完后再返回caller协程继续往下执行
     */
    sc.stop();
```

首先我们看看添加调度任务的部分，这里添加了三个任务，两种方式。一种方式是添加运行函数，包括test_fiber1和test_fiber2，另一种方式是添加协程，包括test_fiber3()，然后这些都是存放在std::list<ScheduleTask> m_tasks任务队列中的。

这两种在调度器中的形式是不同的，但是后面的处理方式是相同的，函数会转化为线程的方式，后面可以会看到。

添加完之后就可以开始运行了，即`sc.start();`。

```c++
void Scheduler::start() {
    SYLAR_LOG_DEBUG(g_logger) << "start";
    MutexType::Lock lock(m_mutex);
    if (m_stopping) {// 已经启动了
        SYLAR_LOG_ERROR(g_logger) << "Scheduler is stopped";
        return;
    }
    SYLAR_ASSERT(m_threads.empty());
    m_threads.resize(m_threadCount);
    for (size_t i = 0; i < m_threadCount; i++) {
        m_threads[i].reset(new Thread(std::bind(&Scheduler::run, this),
                                      m_name + "_" + std::to_string(i)));// 线程执行 run() 任务
        m_threadIds.push_back(m_threads[i]->getId());
    }
}
```

有点意外的是这里用了断言，相当于m_threads一定得为空，那也就是默认协程调度线程也纳入调度器。嗷也不是，就是一开始得是空的，然后会再度修改m_thread的大小这样。

如果不纳入，就给线程初始化这样，然后放入线程池的线程ID数组。

当然，这次运行没有进行到下面。

然后继续添加了函数test_fiber4。

然后就退出了...其实最大的问题是，自己没有感觉到调节的存在...感觉最猛的就是stop函数了。

```c++
void Scheduler::stop() {
    SYLAR_LOG_DEBUG(g_logger) << "stop";
    if (stopping()) {
        return;
    }
    m_stopping = true;// 进入stop将自动停止设为true

    /// 如果use caller，那只能由caller线程发起stop
    if (m_useCaller) {
        SYLAR_ASSERT(GetThis() == this);
    } else {
        SYLAR_ASSERT(GetThis() != this);
    }

    for (size_t i = 0; i < m_threadCount; i++) {// 每个线程都tickle一下
        tickle();
    }

    if (m_rootFiber) {// 使用use_caller多tickle一下
        tickle();
    }

    /// 在use caller情况下，调度器协程结束时，应该返回caller协程
    if (m_rootFiber) {
        m_rootFiber->resume();
        SYLAR_LOG_DEBUG(g_logger) << "m_rootFiber end";
    }

    std::vector<Thread::ptr> thrs;
    {
        MutexType::Lock lock(m_mutex);
        thrs.swap(m_threads);
    }
    for (auto &i : thrs) {// 等待线程执行完成
        i->join();
    }
}
```

其实主要就是运行`m_rootFiber->resume();`切换到调度线程中运行。

然后就是进入MainFunc，然后就是进入run函数。

run函数之后也很多次会用到，不过这个函数出错了...之后再看吧。

然后就是回归到iomanager来了，就是接到初始化完scheduler之后初始化iomanager了。

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

这里很重要的是epoll，pipe，epoll_event这些东西。

首先是`m_epfd = epoll_create(5000); `创建一个epollfd，这是一个库函数，直接调用，返回值m_epfd为文件句柄，即对于打开文件的标识。

这里需要先了解一些关于epoll的东西。

首先epoll是用来做什么的呢，是在多路复用中用到的。通过单线程或单进程同时监测若干个文件描述符是否可以执行IO操作，这就是IO多路复用。

IO多路复用有select，poll，epoll，其中epoll是最新的也是最好的。

这里介绍epoll。epoll有三个需要关注的函数：epoll_create、epoll_ctl、epoll_wait。

epoll的主要数据结构：

```c++
/* 创建epollfd时创建eventpoll结构体，内核维护的就是这个 */
struct eventpoll {
    /* Protect the this structure access */
    spinlock_t lock;
    /* 保证线程安全 */
    struct mutex mtx;
    /* 系统调用等待队列 */
    wait_queue_head_t wq;
    /* epollfd被poll() */
    wait_queue_head_t poll_wait;
    /* 已经ready的epitem */
    struct list_head rdllist;
    /* 所有要监听的epitem的红黑树头节点 */
    struct rb_root rbr;
    /* 这是一个链表，它链接了所有“结构 epitem”
     * epoll_wait()已返回但是又来了新的事件，就保存到这里 */
    struct epitem *ovflist;
    /* 这里保存了一些用户变量 */
    struct user_struct *user;
};

/* epitem 表示一个被监听的fd */
struct epitem {
	/* 所有要监听的epitem红黑树节点 */
	struct rb_node rbn;
	/* 链表节点, 所有已经ready的epitem都会被链到eventpoll的rdllist中 */
	struct list_head rdllink;
	/*
	 * 协同工作 “struct eventpoll”->ovflist 以保持单链接的项目链
	 */
	struct epitem *next;
	/* The file descriptor information this item refers to */
	/* epitem对应的fd和struct file */
	struct epoll_filefd ffd;
	/* 附加到轮询操作的活动等待队列数 */
	int nwait;
	/* 包含轮询等待队列的列表 其实就是被eppoll_entry */
	struct list_head pwqlist;
	/* 当前epitem属于哪个eventpoll */
	struct eventpoll *ep;
	/* 用于将此项目链接到“结构文件”项目列表的列表标题 */
	struct list_head fllink;
	/* 当前的epitem关系哪些events, 这个数据是调用epoll_ctl时从用户态传递过来 */
	struct epoll_event event;
};

/* poll所用到的钩子Wait structure used by the poll hooks */
struct eppoll_entry {
	/* List header used to link this structure to the "struct epitem" */
	struct eppoll_entry *next;
	/* The "base" pointer is set to the container "struct epitem" */
	struct epitem *base;
	/*
	 * Wait queue item that will be linked to the target file wait
	 * queue head.
	 */
	wait_queue_t wait;
	/* The wait queue head that linked the "wait" wait queue item */
	wait_queue_head_t *whead;
};
/* 轮询队列使用的包装器结构 */
struct ep_pqueue {
	poll_table pt;
	struct epitem *epi;
};
/* 保存了文件结构 */
struct epoll_filefd {
	struct file *file;
	int fd;
};
/* ep_send_events()中使用，作为回调函数的参数 */
struct ep_send_events_data {
	int maxevents;
	struct epoll_event __user *events;
};
```

首先是**epoll_create**函数，创建epoll句柄。epollfd对应的不是一个真正通俗意义上的文件，而是内核创建的虚拟文件，然后分配真正的struct file结构, 而且有真正的fd，而eventpoll对象保存在struct file结构的private_data指针中。返回epollfd的文件描述符。

其次是**epoll_ctl**函数，是讲epoll_event结构拷贝到内核空间中。结合使用场景可以知道，这个是自己创造的一个数据，在这个例子中就是管道的读那一端。这里其实是将我们创建的epoll_event放入到主要数据结构eventpoll中。

具体流程：判断加入的`fd`的`f_op->poll`是否支持`poll`结构，通过`epollfd`取得`struct file`中`private_data`获取`eventpoll`，根据`op`区分是添加，修改还是删除。在添加`fd`时，首先在`eventpoll`结构中的红黑树查找是否已经存在了相对应的`epitem`(即是否已经被添加到需要检测的fd中去了),没找到就支持插入操作,否则报重复的错误，并且会调用被监听的`fd`的`poll`方法查看是否有事件发生。

...关于epoll操作，我们新开一篇文章来学习。



然后是`int rt = pipe(m_tickleFds);`创建管道，用于进程间的通信。这里的rt是返回值，只是用来标志是否建立成功，m_tickleFds数组存放的是读文件和写文件的句柄。

关于管道，有两种，一种是匿名的，即子进程和父进程的，这里就是这种的。因为子进程和父进程是同一套东西，所以可以根据各自的fd写入和读取同一个管道文件。为了管理方便，我们关闭了父进程的读以及子进程的写，变为了单项的。

命名管道是不太一样的，在不相关的进程间也可以相互通信，因为命令管道提前创建了一个类型为管道的设备文件，在进程里只要使用这个设备文件，就可以相互通信。

然后进行完这些之后就是`start()`。

```c++
void Scheduler::start() {
    SYLAR_LOG_DEBUG(g_logger) << "start";
    MutexType::Lock lock(m_mutex);
    if (m_stopping) {// 已经启动了
        SYLAR_LOG_ERROR(g_logger) << "Scheduler is stopped";
        return;
    }
    SYLAR_ASSERT(m_threads.empty());
    m_threads.resize(m_threadCount);
    for (size_t i = 0; i < m_threadCount; i++) {
        m_threads[i].reset(new Thread(std::bind(&Scheduler::run, this),m_name + "_" + std::to_string(i)));// 线程执行 run() 任务
        m_threadIds.push_back(m_threads[i]->getId());
    }
}
```

这里因为m_threadCount为0，所以几乎没有做什么操作。

然后进行`iom.schedule(test_io);`。这个其实就是添加需要调度的任务`test_io`，所以其实也没什么操作的含金量mmm。

之后就是主角登场了，`IOManager::~IOManager()`，因为退出了嘛，所以就要执行析构函数。

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

调试后发现，进行完`stop()`之后，后面基本上没啥内容，进行完对IOManager的析构之后，又是对其他很多内容的析构，也没啥内容mmm。所以我们的重点就只有`stop()`这一个函数。

```c++
void Scheduler::stop() {
    SYLAR_LOG_DEBUG(g_logger) << "stop";
    if (stopping()) {
        return;
    }
    m_stopping = true;// 进入stop将自动停止设为true

    /// 如果use caller，那只能由caller线程发起stop
    if (m_useCaller) {
        SYLAR_ASSERT(GetThis() == this);
    } else {
        SYLAR_ASSERT(GetThis() != this);
    }

    for (size_t i = 0; i < m_threadCount; i++) {// 每个线程都tickle一下
        tickle();
    }

    if (m_rootFiber) {// 使用use_caller多tickle一下
        tickle();
    }

    /// 在use caller情况下，调度器协程结束时，应该返回caller协程
    if (m_rootFiber) {
        m_rootFiber->resume();
        SYLAR_LOG_DEBUG(g_logger) << "m_rootFiber end";
    }

    std::vector<Thread::ptr> thrs;
    {
        MutexType::Lock lock(m_mutex);
        thrs.swap(m_threads);
    }
    for (auto &i : thrs) {// 等待线程执行完成
        i->join();
    }
}
```

stop()函数主要运行的是` m_rootFiber->resume();`即调度协程。同样的，执行协程的resume就切换到协程m_rootFiber中去。当然首先去到的是每个fiber的回调函数MainFunc。我们在scheduler初始化的时候知道`m_rootFiber.reset(new Fiber(std::bind(&Scheduler::run, this), 0, false));`进入的是run函数。

```c++
void Scheduler::run() {
    SYLAR_LOG_DEBUG(g_logger) << "run";
    set_hook_enable(true);
    setThis();
    if (sylar::GetThreadId() != m_rootThread) {// 非user_caller线程，设置主协程为线程主协程
        t_scheduler_fiber = sylar::Fiber::GetThis().get();
    }
    // 定义dile_fiber，当任务队列中的任务执行完之后，执行idle()
    Fiber::ptr idle_fiber(new Fiber(std::bind(&Scheduler::idle, this)));//里面其实就是一个构造函数
    Fiber::ptr cb_fiber;

    ScheduleTask task;
    while (true) {
        task.reset();
        bool tickle_me = false; // 是否tickle其他线程进行任务调度
        {// 从任务队列中拿fiber和cb
            MutexType::Lock lock(m_mutex);
            auto it = m_tasks.begin();
            // 遍历所有调度任务
            while (it != m_tasks.end()) {// 如果当前任务指定的线程不是当前线程，则跳过，并且tickle一下
                if (it->thread != -1 && it->thread != sylar::GetThreadId()) {
                    // 指定了调度线程，但不是在当前线程上调度，标记一下需要通知其他线程进行调度，然后跳过这个任务，继续下一个
                    ++it;
                    tickle_me = true;
                    continue;
                }

                // 找到一个未指定线程，或是指定了当前线程的任务
                SYLAR_ASSERT(it->fiber || it->cb);

                // if (it->fiber) {
                //     // 任务队列时的协程一定是READY状态，谁会把RUNNING或TERM状态的协程加入调度呢？
                //     SYLAR_ASSERT(it->fiber->getState() == Fiber::READY);
                // }

                // [BUG FIX]: hook IO相关的系统调用时，在检测到IO未就绪的情况下，会先添加对应的读写事件，再yield当前协程，等IO就绪后再resume当前协程
                // 多线程高并发情境下，有可能发生刚添加事件就被触发的情况，如果此时当前协程还未来得及yield，则这里就有可能出现协程状态仍为RUNNING的情况
                // 这里简单地跳过这种情况，以损失一点性能为代价，否则整个协程框架都要大改
                if(it->fiber && it->fiber->getState() == Fiber::RUNNING) {
                    ++it;
                    continue;
                }
                
                // 当前调度线程找到一个任务，准备开始调度，将其从任务队列中剔除，活动线程数加1
                task = *it;
                m_tasks.erase(it++);
                ++m_activeThreadCount;
                break;
            }
            // 当前线程拿完一个任务后，发现任务队列还有剩余，那么tickle一下其他线程
            tickle_me |= (it != m_tasks.end());
        }

        if (tickle_me) {
            tickle();
        }

        if (task.fiber) {
            // resume协程，resume返回时，协程要么执行完了，要么半路yield了，总之这个任务就算完成了，活跃线程数减一
            task.fiber->resume();
            --m_activeThreadCount;
            task.reset();
        } else if (task.cb) {
            if (cb_fiber) {
                cb_fiber->reset(task.cb);
            } else {
                cb_fiber.reset(new Fiber(task.cb));//相当于调度的是回调函数，也当作协程处理。
            }
            task.reset();
            cb_fiber->resume();
            --m_activeThreadCount;
            cb_fiber.reset();
        } else {
            // 进到这个分支情况一定是任务队列空了，调度idle协程即可
            if (idle_fiber->getState() == Fiber::TERM) {
                // 如果调度器没有调度任务，那么idle协程会不停地resume/yield，不会结束，如果idle协程结束了，那一定是调度器停止了
                SYLAR_LOG_DEBUG(g_logger) << "idle fiber term";
                break;
            }
            ++m_idleThreadCount;
            idle_fiber->resume();
            --m_idleThreadCount;
        }
    }
    SYLAR_LOG_DEBUG(g_logger) << "Scheduler::run() exit";
}
```

run函数循环处理调度任务列表中的任务，一共有三种可能，分别是任务为协程，任务为调度函数，任务为协程，以及没有任务时调用idle_fiber。

因为我们之前添加过任务test_io，所以这里首先执行test_io。对于任务为回调函数的，我们也封装为一个协程，然后MainFunc，然后再运行回调函数。

关于这个回调函数test_io，其实还有点复杂。

```c++
void test_io() {
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    SYLAR_ASSERT(sockfd > 0);
    fcntl(sockfd, F_SETFL, O_NONBLOCK);

    sockaddr_in servaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(1234);
    inet_pton(AF_INET, "10.10.19.159", &servaddr.sin_addr.s_addr);

    int rt = connect(sockfd, (const sockaddr*)&servaddr, sizeof(servaddr));
    if(rt != 0) {
        if(errno == EINPROGRESS) {
            SYLAR_LOG_INFO(g_logger) << "EINPROGRESS";
            // 注册写事件回调，只用于判断connect是否成功
            // 非阻塞的TCP套接字connect一般无法立即建立连接，要通过套接字可写来判断connect是否已经成功
            sylar::IOManager::GetThis()->addEvent(sockfd, sylar::IOManager::WRITE, do_io_write);
            // 注册读事件回调，注意事件是一次性的
            sylar::IOManager::GetThis()->addEvent(sockfd, sylar::IOManager::READ, do_io_read);
        } else {
            SYLAR_LOG_ERROR(g_logger) << "connect error, errno:" << errno << ", errstr:" << strerror(errno);
        }
    } else {
        SYLAR_LOG_ERROR(g_logger) << "else, errno:" << errno << ", errstr:" << strerror(errno);
    }
}
```

这里首先创建了一个IPV4连接，即`sockfd = socket(AF_INET, SOCK_STREAM, 0);`

然后创建了一个IPv4地址，这里相当于是设置为IPv4格式，端口为1234，IP地址为10.10.19.159。

```c++
sockaddr_in servaddr;
memset(&servaddr, 0, sizeof(servaddr));
servaddr.sin_family = AF_INET;
servaddr.sin_port = htons(1234);
inet_pton(AF_INET, "10.10.19.159", &servaddr.sin_addr.s_addr);
```

然后请求连接：`int rt = connect(sockfd, (const sockaddr*)&servaddr, sizeof(servaddr));`

建立连接之后运行addEvent函数。

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
```

首先给这个socket事件创造一个上下文，并放在上下文容器`vector<FdContext *> m_fdContexts`中。这个上下文用fd进行编号以及唯一标识。

之后创造了一个epoll_event结构的epevent，用event(是读还是写，还是五事件)和fd_ctx(放入data指针中)进行初始化，其实此时fd_ctx是空的，后面就会进行初始化。

然后进行注册，用我们熟悉的epoll_ctl函数。

之后对fd_ctx进行初始化。

FdContext的成员变量为：

```C++
/// 读事件上下文
EventContext read;
/// 写事件上下文
EventContext write;
/// 事件关联的句柄
int fd = 0; //可以理解为在数组中是第几个mmm
/// 该fd添加了哪些事件的回调函数，或者说该fd关心哪些事件
Event events = NONE;
/// 事件的Mutex
MutexType mutex;
```

其中EventContext结构体为：

```c++
struct EventContext {
      /// 执行事件回调的调度器
      Scheduler *scheduler = nullptr;
      /// 事件回调协程
      Fiber::ptr fiber;
      /// 事件回调函数
      std::function<void()> cb;
};
```

接下来进行赋值。

```c++
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
```

可以看到选择读或者写事件，然后是事件回调函数还是事件回调协程，这两者都是并列的。

结束之后就回到了MainFunc，然后调用yield回到了run函数中，然后结束此次事件的执行，循环进行下一次。

因为此时m_tasks中已经没有事件了，所以要执行idle协程中的idle函数，注意这里的idle是iomanager中的。

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

这个函数其实非常复杂，下面是他的作用：

调度器无调度任务时会阻塞idle协程上，对IO调度器而言，idle状态应该关注两件事，一是有没有新的调度任务，对应Schduler::schedule()，如果有新的调度任务，那应该立即退出idle状态，并执行对应的任务；二是关注当前注册的所有IO事件有没有触发，如果有触发，那么应该执行IO事件对应的回调函数。

所以其实是在看有没有新的调度任务或者注册IO是否触发。

可以看到函数内主要是一个很大的while(true)循环，结束循环的条件为：

```c++
uint64_t next_timeout = 0;
        if( SYLAR_UNLIKELY(stopping(next_timeout))) {
            SYLAR_LOG_DEBUG(g_logger) << "name=" << getName() << "idle stopping exit";
            break;
        }
```

观察函数stopping：

```c++
bool IOManager::stopping(uint64_t &timeout) {
    // 对于IOManager而言，必须等所有待调度的IO事件都执行完了才可以退出
    // 增加定时器功能后，还应该保证没有剩余的定时器待触发
    timeout = getNextTimer();
    // 定时器为空 && 等待执行的事件数量为0 && scheduler可以stop
    return timeout == ~0ull && m_pendingEventCount == 0 && Scheduler::stopping();
}
```

结束必须满足三个条件：

> 1. 定时器为空
> 2. 等待执行的事件数量为0
> 3. scheduler可以stop

关于scheduler是否可以停止：

```c++
bool Scheduler::stopping() {
    MutexType::Lock lock(m_mutex);// 正在停止 && 任务队列为空 && 活跃的线程数量为0
    return m_stopping && m_tasks.empty() && m_activeThreadCount == 0;
}
```

同样需要满足三个条件：

>1. 正在停止
>2. 任务队列为空
>3. 活跃的线程数量为0

关于函数getNextTimer()：

```c++
uint64_t TimerManager::getNextTimer() {
    RWMutexType::ReadLock lock(m_mutex);
    m_tickled = false;// 不触发 onTimerInsertedAtFront
    if(m_timers.empty()) {// 如果没有定时器，返回一个最大值
        return ~0ull;
    }

    const Timer::ptr& next = *m_timers.begin();// 拿到第一个定时器
    uint64_t now_ms = sylar::GetElapsedMS();
    if(now_ms >= next->m_next) {
        return 0;
    } else {
        return next->m_next - now_ms;
    }
}
```

观察这里timeout的取值会发现，因为m_timers为空，所以就取~0ull，即最大的长整型数。

![image-20240717174824479](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240717174824479.png)

而且在判断条件中为`timeout == ~0ull`，所以相当于这个一定会满足的。

但是之后修改了阻塞事件：

```c++
// 默认超时时间5秒，如果下一个定时器的超时时间大于5秒，仍以5秒来计算超时，避免定时器超时时间太大时，epoll_wait一直阻塞
            static const int MAX_TIMEOUT = 5000;
            if(next_timeout != ~0ull) {
                next_timeout = std::min((int)next_timeout, MAX_TIMEOUT);
            } else {
                next_timeout = MAX_TIMEOUT;
            }//阻塞在这里
```

修改阻塞时间为5s。

然后进行epoll_wait函数。

```c++
rt = epoll_wait(m_epfd, events, MAX_EVNETS, (int)next_timeout);
```

成功时，epoll_wait（）返回为请求的I / O准备就绪的文件描述符的数目，即此时rt大于0。

如果rt大于0，则跳出循环，要知道，epoll_wait是在一个循环里面的，即是在循环里面等待准备就绪的文件。

跳出之后首先处理的就是超时的定时器：

```C++
// 收集所有已超时的定时器，执行回调函数
        std::vector<std::function<void()>> cbs;
        listExpiredCb(cbs);
        if(!cbs.empty()) {
            for(const auto &cb : cbs) {
                schedule(cb);// 全部放到任务队列中
            }
            cbs.clear();
        }
```

不过这里并没有执行。查看cbs的大小可以发现：

![image-20240717200746027](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240717200746027.png)

之后就处理那些准备就绪的文件。

```c++
for (int i = 0; i < rt; ++i) {...}
```

第一种情况，如果是是从我们之前绑定的m_tickleFds[0]来的，那么直接读掉就可以了(这里就是对应我们之前用pipe绑定的部分)。

```c++
if (event.data.fd == m_tickleFds[0]) {// 如果获得的这个信息时来自 pipe
                // ticklefd[0]用于通知协程调度，这时只需要把管道里的内容读完即可
                uint8_t dummy[256];
                while (read(m_tickleFds[0], dummy, sizeof(dummy)) > 0)// 将 pipe 发来的1个字节数据读掉
                    ;
                continue;
            }
```

剩下就是socket的了，这里又分为两种情况，一种是读，另一种是写。

```c++
int real_events = NONE;
            if (event.events & EPOLLIN) {// 读事件好了
                real_events |= READ;
            }
            if (event.events & EPOLLOUT) {// 写事件好了
                real_events |= WRITE;
            }
```

首先确定是读事件还是写事件。

```c++
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
```

需要做的事情剔除之后，重新注册一下。

```c++
int left_events = (fd_ctx->events & ~real_events);
int op          = left_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;// 如果执行完该事件还有事件则修改，若无事件则删除
event.events    = EPOLLET | left_events;// 更新新的事件

int rt2 = epoll_ctl(m_epfd, op, fd_ctx->fd, &event);// 重新注册事件
```

然后处理读事件或者是写事件。

```c++
// 处理已经发生的事件，也就是让调度器调度指定的函数或协程
if (real_events & READ) {// 读事件好了，执行读事件
   fd_ctx->triggerEvent(READ);
                --m_pendingEventCount;
            }
            if (real_events & WRITE) {// 写事件好了，执行写事件
                fd_ctx->triggerEvent(WRITE);
                --m_pendingEventCount;
            }
```

看一下处理的triggerEvent函数。

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

可以看到，其实是将需要做的回调函数和协程放到任务队列里面。

然后`--m_pendingEventCount`，表示等待执行的IO事件数量-1，每次我们注册的时候都会进行`++m_pendingEventCount`。

当全部执行完之后，运行`raw_ptr->yield();`回到idle_fiber->resume();下面的位置。

之后退出此次循环，开启下一次循环。

可以看到此时m_tasks的大小为2。

![image-20240717210343666](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240717210343666.png)

运行发现进入了`do_io_read`函数。

```c++
void do_io_read() {
    SYLAR_LOG_INFO(g_logger) << "read callback";
    char buf[1024] = {0};
    int readlen = 0;
    readlen = read(sockfd, buf, sizeof(buf));
    if(readlen > 0) {
        buf[readlen] = '\0';
        SYLAR_LOG_INFO(g_logger) << "read " << readlen << " bytes, read: " << buf;
    } else if(readlen == 0) {
        SYLAR_LOG_INFO(g_logger) << "peer closed";
        close(sockfd);
        return;
    } else {
        SYLAR_LOG_ERROR(g_logger) << "err, errno=" << errno << ", errstr=" << strerror(errno);
        close(sockfd);
        return;
    }
    // read之后重新添加读事件回调，这里不能直接调用addEvent，因为在当前位置fd的读事件上下文还是有效的，直接调用addEvent相当于重复添加相同事件
    sylar::IOManager::GetThis()->schedule(watch_io_read);
}
```

不过这里的返回值readlen为-1，error了，很尴尬，然后后面watch_io_read也没有正常添加。自己也不知道应该不应该是这样的。

然后就是下面一轮进行了do_io_write函数。

```c++
void do_io_write() {
    SYLAR_LOG_INFO(g_logger) << "write callback";
    int so_err;
    socklen_t len = size_t(so_err);
    getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &so_err, &len);
    if(so_err) {
        SYLAR_LOG_INFO(g_logger) << "connect fail";
        return;
    } 
    SYLAR_LOG_INFO(g_logger) << "connect success";
}
```

然后这个也fail了...更尴尬了。

之后回到run中，因为此时m_tasks中没有需要做的任务了，所以又进入了`idle_fiber->resume();`中了。

因为之前运行过，所以这次回到了上次yield的下面了，然后再次进行循环，这次退出循环了，通过yield回去，回到了`idle_fiber->resume();`下面。

这次再次进入循环之后，真正的退出循环了。

```c++
if (idle_fiber->getState() == Fiber::TERM) {
                // 如果调度器没有调度任务，那么idle协程会不停地resume/yield，不会结束，如果idle协程结束了，那一定是调度器停止了
                SYLAR_LOG_DEBUG(g_logger) << "idle fiber term";
                break;
            }
```

总结一下吧，然后就要开启下一个部分了。大概明白运行的逻辑了，但是涉及到epoll_wait的部分还是不太明白。

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

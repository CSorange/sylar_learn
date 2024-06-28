# sylar学习06：协程调度模块

在进行之前，自己思考了一下应该如何写，如果组织行文结构。其实自己学了这么一段时间，大概明白了有两种方式，一种是从主函数出发，遇到什么就分析什么，这是自顶向下。还有一种是从类出发，自底向上。都尝试过，但发现都没有很顺畅，所以之后记录的时候就先自底向上，再自顶向下。有一个问题是有的结构要写两遍，现在想来感觉也没什么。

关于协程调度模块：

> 封装了一个N : M协程调度器，创建M个协程在N个线程上运行。通过`schedule()`方法将`cb或fiber`重新加到任务队列中执行任务，协程可以在线程上自由切换，也可以在指定线程上执行。

首先构造了一个含有N个线程的线程池，之后构建了一个协程调度器，关于协程与线程的对应有两种方式：一种是随机选择空闲的协程执行，一种是指定在某个线程上执行。

两个局部线程变量：

```c++
/// 当前线程的调度器，同一个调度器下的所有线程共享同一个实例
static thread_local Scheduler *t_scheduler = nullptr;
/// 当前线程的调度协程，每个线程都独有一份
static thread_local Fiber *t_scheduler_fiber = nullptr;
```

调度器**Scheduler**中嵌套一个结构体**ScheduleTask**

```c++
struct ScheduleTask {
        Fiber::ptr fiber;// 协程
        std::function<void()> cb;// 协程执行函数
        int thread;// 线程id 协程在哪个线程上

        ScheduleTask(Fiber::ptr f, int thr) {// 确定协程在哪个线程上跑
            fiber  = f;
            thread = thr;
        }
        ScheduleTask(Fiber::ptr *f, int thr) {// 通过swap将传入的 fiber 置空，使其引用计数-1
            fiber.swap(*f);
            thread = thr;
        }
        ScheduleTask(std::function<void()> f, int thr) {// 确定回调在哪个线程上跑
            cb     = f;
            thread = thr;
        }
        ScheduleTask() { thread = -1; }// 默认构造

        void reset() {// 重置
            fiber  = nullptr;
            cb     = nullptr;
            thread = -1;
        }
    };
```

可以看出，**Scheduler**通过**ScheduleTask**的结构来储存协程，回调函数，线程之间的交互信息。

成员函数：

```c++
/// 协程调度器名称
std::string m_name;
/// 互斥锁
MutexType m_mutex;
/// 线程池
std::vector<Thread::ptr> m_threads;
/// 任务队列
std::list<ScheduleTask> m_tasks;
/// 线程池的线程ID数组
std::vector<int> m_threadIds;
/// 工作线程数量，不包含use_caller的主线程
size_t m_threadCount = 0;
/// 活跃线程数
std::atomic<size_t> m_activeThreadCount = {0};
/// idle线程数
std::atomic<size_t> m_idleThreadCount = {0};

/// 是否use caller
bool m_useCaller;
/// use_caller为true时，调度器所在线程的调度协程
Fiber::ptr m_rootFiber;
/// use_caller为true时，调度器所在线程的id
int m_rootThread = 0;

/// 是否正在停止
bool m_stopping = false;
```

调度协程：

```c++
/**
     * @brief 添加调度任务
     * @tparam FiberOrCb 调度任务类型，可以是协程对象或函数指针
     * @param[] fc 协程对象或指针
     * @param[] thread 指定运行该任务的线程号，-1表示任意线程
     */
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

可以看到，我们在加锁之后，就运行函数`scheduleNoLock`。

```c++
/**
     * @brief 添加调度任务，无锁
     * @tparam FiberOrCb 调度任务类型，可以是协程对象或函数指针
     * @param[] fc 协程对象或指针
     * @param[] thread 指定运行该任务的线程号，-1表示任意线程
     */
    template <class FiberOrCb>
    bool scheduleNoLock(FiberOrCb fc, int thread) {
        bool need_tickle = m_tasks.empty();
        ScheduleTask task(fc, thread);
        if (task.fiber || task.cb) {
            m_tasks.push_back(task);
        }
        return need_tickle;
    }
```

将调度任务指定分配给线程thread执行，创建一个task来表示并将其加入到任务列表m_tasks上去。need_tickle判断原来任务列表上有无任务。如果原来任务队列本来没有任务，就tickle一下。(这一点好像参考的博客写错了)。

构造函数：

```c++
Scheduler::Scheduler(size_t threads, bool use_caller, const std::string &name) {
    SYLAR_ASSERT(threads > 0);// 确定线程数量要正确

    m_useCaller = use_caller; 
    m_name      = name;

    if (use_caller) {// 是否将协程调度线程也纳入调度器
        --threads;
        sylar::Fiber::GetThis();// 获得主协程
        SYLAR_ASSERT(GetThis() == nullptr);
        t_scheduler = this; // 设置当前协程调度器

        /**
         * caller线程的主协程不会被线程的调度协程run进行调度，而且，线程的调度协程停止时，应该返回caller线程的主协程
         * 在user caller情况下，把caller线程的主协程暂时保存起来，等调度协程结束时，再resume caller协程
         */
        m_rootFiber.reset(new Fiber(std::bind(&Scheduler::run, this), 0, false));
        sylar::Thread::SetName(m_name);
        //设置当前线程的主协程为m_rootFiber，这里的m_rootFiber是该线程的主协程（执行run任务的协程），只有默认构造出来的fiber才是主协程
        t_scheduler_fiber = m_rootFiber.get();
        m_rootThread      = sylar::GetThreadId();// 获得当前线程id
        m_threadIds.push_back(m_rootThread);
    } else {// 不将当前线程纳入调度器
        m_rootThread = -1;
    }
    m_threadCount = threads;
}
```

在设计协程调度器时，设置了一个`use_caller`来决定是否将当前调度线程也纳入调度中，这样可以少创建一个线程执行任务，效率更高。构造函数就是在进行开启`use_caller`后操作。

不过这里的reset函数自己确实没有看懂mmm。

析构函数：

```c++
Scheduler::~Scheduler() {
    SYLAR_LOG_DEBUG(g_logger) << "Scheduler::~Scheduler()";
    SYLAR_ASSERT(m_stopping);
    if (GetThis() == this) {
        t_scheduler = nullptr;
    }
}
```

这里直接结束了，感觉还能看懂一点mmm

启动调度器：

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

可以看到，其实reset函数才是主要的运行函数。

停止调度器：

```c++
void Scheduler::stop() {
    SYLAR_LOG_DEBUG(g_logger) << "stop";
    if (stopping()) {
        return;
    }
    m_stopping = true;

    /// 如果use caller，那只能由caller线程发起stop
    if (m_useCaller) {
        SYLAR_ASSERT(GetThis() == this);
    } else {
        SYLAR_ASSERT(GetThis() != this);
    }

    for (size_t i = 0; i < m_threadCount; i++) {
        tickle();
    }

    if (m_rootFiber) {
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
    for (auto &i : thrs) {
        i->join();
    }
}
```

run(协程调度函数)

```c++
void Scheduler::run() {
    SYLAR_LOG_DEBUG(g_logger) << "run";
    set_hook_enable(true);
    setThis();
    if (sylar::GetThreadId() != m_rootThread) {// 非user_caller线程，设置主协程为线程主协程
        t_scheduler_fiber = sylar::Fiber::GetThis().get();
    }
    // 定义dile_fiber，当任务队列中的任务执行完之后，执行idle()
    Fiber::ptr idle_fiber(new Fiber(std::bind(&Scheduler::idle, this)));
    Fiber::ptr cb_fiber;

    ScheduleTask task;
    while (true) {
        task.reset();
        bool tickle_me = false; // 是否tickle其他线程进行任务调度
        {// 从任务队列中拿fiber和cb
            MutexType::Lock lock(m_mutex);
            auto it = m_tasks.begin();
            // 遍历所有调度任务
            while (it != m_tasks.end()) {
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
                cb_fiber.reset(new Fiber(task.cb));
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

我感觉大概能看懂这个函数是用来做什么的了。首先是因为这个函数一定运行在一个线程内。我们需要遍历调度队列，寻找需要在这个线程解决的任务，然后解决他(不过在函数内没有看到具体解决他的部分)。

之后就是一系列处理，不过自己对于协程的处理不是很明白，所以看不太懂。

stopping(判断停止条件)

```c++
bool Scheduler::stopping() {
    MutexType::Lock lock(m_mutex);// 正在停止 && 任务队列为空 && 活跃的线程数量为0
    return m_stopping && m_tasks.empty() && m_activeThreadCount == 0;
}
```


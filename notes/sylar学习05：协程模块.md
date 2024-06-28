# sylar学习05：协程模块

我们知道，进程，线程，协程为三个层次。



> **从定义来看**：
>
> 进程是资源分配和拥有的基本单位。进程通过内存映射拥有独立的代码和数据空间，若没有内存映射给进程独立的空间，则没有进程的概念了。
>
> 线程是程序执行的基本单位。线程都处在一个进程空间中，可以相互访问，没有限制，所以使用线程进行多任务变成十分便利，所以当一个线程崩溃，其他任何一个线程都不能幸免。每个进程中都有唯一的主线程，且只能有一个，主线程和进程是相互依存的关系，主线程结束进程也会结束。
>
> 协程是用户态的轻量级线程，线程内部调度的基本单位。协程在线程上执行。
>
> **从系统调用来看**：
>
> 进程由操作系统进行切换，会在用户态与内核态之间来回切换。在切换进程时需要切换虚拟内存空间，切换页表，切换内核栈以及硬件上下文等，开销非常大。
>
> 线程由操作系统进行切换，会在用户态与内核态之间来回切换。在切换线程时需要保存和设置少量寄存器内容，开销很小。
>
> 协程由用户进行切换，并不会陷入内核态。先将寄存器上下文和栈保存，等切换回来的时候再进行恢复，上下文的切换非常快
>
> **从并发性来看**：
>
> 不同进程之间切换实现并发，各自占有CPU实现并行
>
> 一个进程内部的多个线程并发执行
>
> 同一时间只能执行一个协程，而其他协程处于休眠状态，适合对任务进行分时处理

协程是为了更好的完成任务，提供除了从头到尾之外更多的可能，并不是为了并发性而存在的，而是为了更好的完成一个线程内的任务。线程更多承载的是并发性，进程更多承载的是封装，各有特色吧。

协程模块并没有自己进行实现，而是调用库函数中的`ucontext.h`。

```c++
typedef struct ucontext_t
  {
    unsigned long int __ctx(uc_flags);
    struct ucontext_t *uc_link;
    stack_t uc_stack;
    mcontext_t uc_mcontext;
    sigset_t uc_sigmask;
    struct _libc_fpstate __fpregs_mem;
    __extension__ unsigned long long int __ssp[4];
  } ucontext_t;
```

`uc_link`为此协程运行完之后要进行的协程(可见协程是串行运行的)。`uc_sigmask`：执行当前上下文过程中需要屏蔽的信号列表，即信号掩码。`uc_stack`：为当前`context`运行的栈信息。

```c++
typedef struct
  {
    void *ss_sp;
    int ss_flags;
    size_t ss_size;
  } stack_t;
```

 `ss_sp`：栈指针指向`stack`。`uc_stack.ss_sp = stack`; 

`ss_size`：栈大小。`uc_stack.ss_size = stacksize;`

`uc_mcontext`：保存具体的程序执行上下文，如PC值，堆栈指针以及寄存器值等信息。它的实现依赖于底层，是平台硬件相关的。此实现不透明。

库函数中的`ucontext.h`还有几个相对应的库函数。

```c++
void makecontext(ucontext_t* ucp, void (*func)(), int argc, ...);
int swapcontext(ucontext_t* olducp, ucontext_t* newucp);
int getcontext(ucontext_t* ucp);
int setcontext(const ucontext_t* ucp);
```

**makecontext**：初始化一个ucontext_t，func参数指明了该context的入口函数，argc为入口参数的个数，每个参数的类型必须是int类型。另外在makecontext前，一般需要显示的初始化栈信息以及信号掩码集同时也需要初始化uc_link，以便程序退出上下文后继续执行。

**swapcontext**：原子操作，该函数的工作是保存当前上下文并将上下文切换到新的上下文运行。

**getcontext**：将当前的执行上下文保存在cpu中，以便后续恢复上下文。

**setcontext**：将当前程序切换到新的context,在执行正确的情况下该函数直接切换到新的执行状态，不会返回。

关于协程之间的交互：(同样是调用的库函数，在工程中并没有对应的代码)

>使用非对称协程的设计思路，通过主协程创建新协程，主协程由`swapIn()`让出执行权执行子协程的任务，子协程可以通过`YieldToHold()`让出执行权继续执行主协程的任务，不能在子协程之间做相互的转化，这样会导致回不到`main`函数的上下文。这里使用了两个线程局部变量保存当前协程和主协程，切换协程时调用`swapcontext`，若两个变量都保存子协程，则无法回到原来的主协程中。

不过我其实没有很懂mmm，上面不是每个协程都有下一个应该运行的协程么，为什么还需要主协程，子协程呢，直接串行运行不就好了。

```c++
/// 全局静态变量，用于生成协程id
static std::atomic<uint64_t> s_fiber_id{0};
/// 全局静态变量，用于统计当前的协程数
static std::atomic<uint64_t> s_fiber_count{0};

/// 线程局部变量，当前线程正在运行的协程
static thread_local Fiber *t_fiber = nullptr;
/// 线程局部变量，当前线程的主协程，切换到这个协程，就相当于切换到了主线程中运行，智能指针形式
static thread_local Fiber::ptr t_thread_fiber = nullptr;

//协程栈大小，可通过配置文件获取，默认128k
static ConfigVar<uint32_t>::ptr g_fiber_stack_size =
    Config::Lookup<uint32_t>("fiber.stack_size", 128 * 1024, "fiber stack size");
```

哦对，如果是在协程中运行，其实我们需要保留线程的特征，所以其实主协程就是线程。

```c++
static void *Alloc(size_t size) { return malloc(size); }
static void Dealloc(void *vp, size_t size) { return free(vp); }
```

创建/释放协程运行栈，即占用的空间。

协程有三种状态：

```c++
enum State {
        /// 就绪态，刚创建或者yield之后的状态
        READY,
        /// 运行态，resume之后的状态
        RUNNING,
        /// 结束态，协程的回调函数执行完之后为TERM状态
        TERM
    };
```

这个和自己参考的博客不太一样...他那边有五种状态：*// 初始化*   INIT； *// 暂停*   HOLD；*// 执行*   EXEC；*// 结束*   TERM；*// 可执行*   READY；*// 异常*   EXCEPT。

所包含的成员有：

```c++
/// 协程id
uint64_t m_id        = 0;
/// 协程栈大小
uint32_t m_stacksize = 0;
/// 协程状态
State m_state        = READY;
/// 协程上下文
ucontext_t m_ctx;
/// 协程栈地址
void *m_stack = nullptr;
/// 协程入口函数
std::function<void()> m_cb;
/// 本协程是否参与调度器调度
bool m_runInScheduler;
```

关于构造函数，一共有两个，一个是在private中不带参数的构造函数，一个是在public中带参数的构造函数。其中前者是初始化主协程，后者是初始化子协程。

```c++
Fiber::Fiber() {
    SetThis(this); //获取当前协程
    m_state = RUNNING;

    if (getcontext(&m_ctx)) { //获取当前协程的上下文信息保存到m_ctx中
        SYLAR_ASSERT2(false, "getcontext");
    }

    ++s_fiber_count;
    m_id = s_fiber_id++; // 协程id从0开始，用完加1

    SYLAR_LOG_DEBUG(g_logger) << "Fiber::Fiber() main id = " << m_id;
}
```

主协程是没有栈的，下面的子协程才有栈，而且主协程没有cb，感觉主协程似乎也不做什么，就是起到一个管理的作用。感觉没有做什么就初始化完成了，难道线程初始化之后就由一个子协程作为主协程么。

```c++
Fiber::Fiber(std::function<void()> cb, size_t stacksize, bool run_in_scheduler)
    : m_id(s_fiber_id++)
    , m_cb(cb)
    , m_runInScheduler(run_in_scheduler) {
    ++s_fiber_count;
    m_stacksize = stacksize ? stacksize : g_fiber_stack_size->getValue();// 若给了初始化值则用给定值，若没有则用约定值
    m_stack     = StackAllocator::Alloc(m_stacksize);// 获得协程运行指针

    if (getcontext(&m_ctx)) {// 保存当前协程上下文信息到m_ctx中
        SYLAR_ASSERT2(false, "getcontext");
    }

    m_ctx.uc_link          = nullptr;// uc_link为空，执行完当前context之后退出程序。
    m_ctx.uc_stack.ss_sp   = m_stack;// 初始化栈指针
    m_ctx.uc_stack.ss_size = m_stacksize;// 初始化栈大小

    makecontext(&m_ctx, &Fiber::MainFunc, 0); // 指明该context入口函数

    SYLAR_LOG_DEBUG(g_logger) << "Fiber::Fiber() id = " << m_id;
}
```

析构函数：

```c++
Fiber::~Fiber() {
    SYLAR_LOG_DEBUG(g_logger) << "Fiber::~Fiber() id = " << m_id;
    --s_fiber_count;
    if (m_stack) {
        // 有栈，说明是子协程，需要确保子协程一定是结束状态
        SYLAR_ASSERT(m_state == TERM);
        StackAllocator::Dealloc(m_stack, m_stacksize);
        SYLAR_LOG_DEBUG(g_logger) << "dealloc stack, id = " << m_id;
    } else {
        // 没有栈，说明是线程的主协程
        SYLAR_ASSERT(!m_cb);              // 主协程没有cb
        SYLAR_ASSERT(m_state == RUNNING); // 主协程一定是执行状态

        Fiber *cur = t_fiber; // 当前协程就是自己
        if (cur == this) {
            SetThis(nullptr);
        }
    }
}
```

reset(重置协程)：

```c++
/**
 * 这里为了简化状态管理，强制只有TERM状态的协程才可以重置，但其实刚创建好但没执行过的协程也应该允许重置的
 */
void Fiber::reset(std::function<void()> cb) {
    SYLAR_ASSERT(m_stack);
    SYLAR_ASSERT(m_state == TERM);
    m_cb = cb;
    if (getcontext(&m_ctx)) {
        SYLAR_ASSERT2(false, "getcontext");
    }

    m_ctx.uc_link          = nullptr;
    m_ctx.uc_stack.ss_sp   = m_stack;
    m_ctx.uc_stack.ss_size = m_stacksize;

    makecontext(&m_ctx, &Fiber::MainFunc, 0);
    m_state = READY;
}
```

resume(切换到当前进程)

```c++
void Fiber::resume() {
    SYLAR_ASSERT(m_state != TERM && m_state != RUNNING);
    SetThis(this);
    m_state = RUNNING;

    // 如果协程参与调度器调度，那么应该和调度器的主协程进行swap，而不是线程主协程
    if (m_runInScheduler) {
        if (swapcontext(&(Scheduler::GetMainFiber()->m_ctx), &m_ctx)) {
            SYLAR_ASSERT2(false, "swapcontext");
        }
    } else {
        if (swapcontext(&(t_thread_fiber->m_ctx), &m_ctx)) {
            SYLAR_ASSERT2(false, "swapcontext");
        }
    }
}
```

这个函数相当于把参考博客中的call函数和swapIn函数进行了合并。如果是从调度器的主协程切换到当前协程，则进入if，如果是从协程主协程切换到当前协程，则进入else。(不过目前自己还不太理解这两者之间的区别)。

```c++
void Fiber::yield() {
    /// 协程运行完之后会自动yield一次，用于回到主协程，此时状态已为结束状态
    SYLAR_ASSERT(m_state == RUNNING || m_state == TERM);
    SetThis(t_thread_fiber.get());
    if (m_state != TERM) {
        m_state = READY;
    }

    // 如果协程参与调度器调度，那么应该和调度器的主协程进行swap，而不是线程主协程
    if (m_runInScheduler) {
        if (swapcontext(&m_ctx, &(Scheduler::GetMainFiber()->m_ctx))) {
            SYLAR_ASSERT2(false, "swapcontext");
        }
    } else {
        if (swapcontext(&m_ctx, &(t_thread_fiber->m_ctx))) {
            SYLAR_ASSERT2(false, "swapcontext");
        }
    }
}
```

当前协程切换到后台，同样的合并了back和swapOut两个函数。



之后我们探究`test_fiber.cc`函数。

大体一看，其实和上一章节中的格式相差不大，都是创建日志，处理命令行，读入配置文件，然后创建线程。不同的是，这次的回调函数cb是对于线程的处理。我们直接看与之前有所不同的回调函数，即`test_fiber`。

```c++
void test_fiber() {
    SYLAR_LOG_INFO(g_logger) << "test_fiber begin";

    // 初始化线程主协程
    sylar::Fiber::GetThis();

    sylar::Fiber::ptr fiber(new sylar::Fiber(run_in_fiber, 0, false));
    SYLAR_LOG_INFO(g_logger) << "use_count:" << fiber.use_count(); // 1

    SYLAR_LOG_INFO(g_logger) << "before test_fiber resume";
    fiber->resume();
    SYLAR_LOG_INFO(g_logger) << "after test_fiber resume";

    /** 
     * 关于fiber智能指针的引用计数为3的说明：
     * 一份在当前函数的fiber指针，一份在MainFunc的cur指针
     * 还有一份在在run_in_fiber的GetThis()结果的临时变量里
     */
    SYLAR_LOG_INFO(g_logger) << "use_count:" << fiber.use_count(); // 3

    SYLAR_LOG_INFO(g_logger) << "fiber status: " << fiber->getState(); // READY

    SYLAR_LOG_INFO(g_logger) << "before test_fiber resume again";
    fiber->resume();
    SYLAR_LOG_INFO(g_logger) << "after test_fiber resume again";

    SYLAR_LOG_INFO(g_logger) << "use_count:" << fiber.use_count(); // 1
    SYLAR_LOG_INFO(g_logger) << "fiber status: " << fiber->getState(); // TERM

    fiber->reset(run_in_fiber2); // 上一个协程结束之后，复用其栈空间再创建一个新协程
    fiber->resume();

    SYLAR_LOG_INFO(g_logger) << "use_count:" << fiber.use_count(); // 1
    SYLAR_LOG_INFO(g_logger) << "test_fiber end";
}
```

可以看到，我们其实是用GetThis函数来行使创建当前线程主协程的作用。如果当前运行协程t_fiber不为空，则返回这个协程，如果为空，则创建一个主协程。

```c++
Fiber::ptr Fiber::GetThis() {
    if (t_fiber) {//返回当前协程
        return t_fiber->shared_from_this();
    }
    //获得主协程
    Fiber::ptr main_fiber(new Fiber);
    SYLAR_ASSERT(t_fiber == main_fiber.get());//此时当前协程为主协程
    t_thread_fiber = main_fiber;
    return t_fiber->shared_from_this();
}
```

这里我们使用的是主协程的创建，即不带参数的初始化。

```c++
Fiber::Fiber() {
    SetThis(this); //获取当前协程
    m_state = RUNNING;

    if (getcontext(&m_ctx)) { //获取当前协程的上下文信息保存到m_ctx中
        SYLAR_ASSERT2(false, "getcontext");
    }

    ++s_fiber_count;
    m_id = s_fiber_id++; // 协程id从0开始，用完加1

    SYLAR_LOG_DEBUG(g_logger) << "Fiber::Fiber() main id = " << m_id;
}
```

不过这里自己不是很明白的是第一行，获取当前协程。不是还没有创建么，为什么可以获取到呢。调试发现在执行之前其实就有this指针了。

![image-20240627210119876](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240627210119876.png)

```c++
void Fiber::SetThis(Fiber *f) { t_fiber = f; }
static thread_local Fiber *t_fiber = nullptr;
```

创建主协程并将主协程设置为当前正在运行的协程。

之后把生成的主协程main_thread赋值给静态变量t_thread_fiber并返回即可。

在创建完主协程之后，就开始创建子协程：(此时仍然在当前线程中，或者说这整个流程都在当前线程中)。

`sylar::Fiber::ptr fiber(new sylar::Fiber(run_in_fiber, 0, false));`

子协程的创建便是用的带参数的构造函数。

```c++
Fiber::Fiber(std::function<void()> cb, size_t stacksize, bool run_in_scheduler)
    : m_id(s_fiber_id++)
    , m_cb(cb)
    , m_runInScheduler(run_in_scheduler) {
    ++s_fiber_count;
    m_stacksize = stacksize ? stacksize : g_fiber_stack_size->getValue();// 若给了初始化值则用给定值，若没有则用约定值
    m_stack     = StackAllocator::Alloc(m_stacksize);// 获得协程运行指针

    if (getcontext(&m_ctx)) {// 保存当前协程上下文信息到m_ctx中
        SYLAR_ASSERT2(false, "getcontext");
    }

    m_ctx.uc_link          = nullptr;// uc_link为空，执行完当前context之后退出程序。
    m_ctx.uc_stack.ss_sp   = m_stack;// 初始化栈指针
    m_ctx.uc_stack.ss_size = m_stacksize;// 初始化栈大小

    makecontext(&m_ctx, &Fiber::MainFunc, 0); // 指明该context入口函数

    SYLAR_LOG_DEBUG(g_logger) << "Fiber::Fiber() id = " << m_id;
}
```

这里我们把子协程建好之后，就通过makecontext函数进行初始化。这里同样有一个回调函数，即MainFunc函数。

```c++
void Fiber::MainFunc() {
    Fiber::ptr cur = GetThis(); // 获得当前进程，GetThis()的shared_from_this()方法让引用计数加1
    SYLAR_ASSERT(cur);
    //执行任务
    cur->m_cb();
    cur->m_cb    = nullptr;
    cur->m_state = TERM;// 将状态设置为结束

    auto raw_ptr = cur.get(); // 手动让t_fiber的引用计数减1
    cur.reset();
    raw_ptr->yield(); //执行完释放执行权
}
```

这里执行了m_cb，即通过子协程构造函数传入的run_in_fiber函数进行对协程的操作。

但这里自己其实没有很明白关于协程切换的操作，主要不太能知道协程如何切换这样，之后再看看吧。
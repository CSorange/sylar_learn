# sylar学习10：线程调试

由于之前协程调试的过程有很多地方没有搞清楚，所以在这里再看一下。这次调试用到了一些gdb命令：

1. 只让当前线程运行：`set scheduler-locking on`；于此对应的有

不只让当前线程运行：`set scheduler-locking off`。

2. 切换线程：thread x。这个自己可以多用，之前没有用过。

我们以`test_fiber.cc`为例进行分析。

涉及到线程生成的主体部分为：

```C++
std::vector<sylar::Thread::ptr> thrs;
    for (int i = 0; i < 2; i++) {
        thrs.push_back(sylar::Thread::ptr(
            new sylar::Thread(&test_fiber, "thread_" + std::to_string(i))));
    }

    for (auto i : thrs) {
        i->join();
    }
```

运行到`new sylar::Thread...`这一行的时候，我们进入thread的构造函数：

```c++
Thread::Thread(std::function<void()> cb, const std::string &name)
    : m_cb(cb)
    , m_name(name) {
    if (name.empty()) {
        m_name = "UNKNOW";
    }
    int rt = pthread_create(&m_thread, nullptr, &Thread::run, this);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger) << "pthread_create thread fail, rt=" << rt
                                  << " name=" << name;
        throw std::logic_error("pthread_create error");
    }
    m_semaphore.wait();
}
```

可以看到最主要的函数为线程生成函数`pthread_create`。实际在调试这个部分的时候会有些困难，因为涉及到库函数中的代码，所以很难看到在`pthread_create`中进行到哪一步了，也就没有办法判断该进行n还是s，一不小心会跳过很关键的地方。

多尝试几次后发现了规律：在`pthread_create`函数中一开始可以大跨步的走，即用n，当运行到大概770行左右需要放慢速度，函数会进入子函数`createthread.c`，之后会进入`clone.S`，更靠近底层的线程复制函数，然后我们一步步s，直到看到`[New Thread 0x7ffff772d700 (LWP 40853)]`，这表示新的线程生成了，这时候我们赶紧输入`set scheduler-locking on`，把新生成的线程摁住不让他运行。之后就可以大跨步的n，一直到当前主线程跳出`pthread_create`函数。之后就可以对生成的子线程进行调试了。

这时候我们查看线程的话会发现：

![image-20240701155730555](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240701155730555.png)

其实子线程就是父线程的一个克隆体，很多地方都是一样的(最典型的就是名字了)。

emmm以上感觉还是不太靠谱，所以我采用了另一个办法，就是请过程，重关键时刻。

当进行到线程生成函数`pthread_create`时，直接在void *Thread::run处打断点，直接让程序运行到这个地方。

此时线程已经复制，需要调用run进行对子线程进行处理。可以看此时的线程：

![image-20240701161929675](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240701161929675.png)

因为run函数是在子线程中进行的嘛，如果我们切换到主线程进行s，程序会卡死。原因是现在主线程卡住了，在等待子线程运行到相应的位置。

观察函数`futex_abstimed_wait_cancelable`也可以看出。

之后继续运行run函数。

```c++
void *Thread::run(void *arg) {
    Thread *thread = (Thread *)arg; // 拿到新创建的Thread对象
    t_thread       = thread; // 更新当前线程
    t_thread_name  = thread->m_name;
    thread->m_id   = sylar::GetThreadId(); // 设置当前线程的id
    // 只有进了run方法才是新线程在执行，创建时是由主线程完成的，threadId为主线程的
    pthread_setname_np(pthread_self(), thread->m_name.substr(0, 15).c_str());  // 设置线程名称
    // pthread_creat时返回引用 防止函数有智能指针,
    std::function<void()> cb;
    cb.swap(thread->m_cb);
    // 在出构造函数之前，确保线程先跑起来，保证能够初始化id
    thread->m_semaphore.notify();
    cb();
    return 0;
}
```

可以看到这里是对子线程的修饰，例如修改子线程的名字。最重要的就是cb()回调函数了，这是从`test_fiber.cc`文件中一路传进来的，即`test_fiber`函数。

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
    ...
```

可以看到获得当前协程函数`GetThis()`，其实在这里才涉及到协程的部分。

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

我们一般用这个函数来初始化主协程，当然初始化主协程时还没有协程，如果当前有协程，那么这个函数的功能就是返回当前协程。

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

这里其实感官上和线程差别比较大，因为没有类似thread_create函数这样。可能协程就是比较简单，只是顺序执行，没有那么多幺蛾子，需要锁之类的东西。主要依赖的是上下文，这部分倒是主函数控制的。

```c++
#include <ucontext.h>
typedef struct ucontext_t {
  struct ucontext_t* uc_link;//当前context执行结束之后要执行的下一个context
  sigset_t uc_sigmask;//执行当前上下文过程中需要屏蔽的信号列表
  stack_t uc_stack;//为当前context运行的栈信息。
  mcontext_t uc_mcontext;//保存具体的程序执行上下文，如PC值，堆栈指针以及寄存器值等信息。
  ...
};
```

搞定主协程之后(当前协程)，之后就要生成子协程。

```C++
sylar::Fiber::ptr fiber(new sylar::Fiber(run_in_fiber, 0, false));
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

之后进行resume函数。

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

这里的setThis函数执行这样的切换，因为是进行`fiber->resume();`所以是切换到fiber了，即this表示的是fiber。

**swapcontext**函数是原子操作，该函数的工作是保存当前上下文并将上下文切换到新的上下文运行。自己理解错了，还以为仅仅是交换。

因为之前在fiber子协程的构造函数中，我们进行了操作：

`makecontext(&m_ctx, &Fiber::MainFunc, 0); // 指明该context入口函数`，所以这里的swapcontext是保存当前上下文，然后进入MainFunc函数。

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
    raw_ptr->yield();
}
```

可以看到这里需要运行cb回调函数。其实就是在生成子协程的回调函数绕了很远，才在resume函数中的MainFunc函数中运行。

至于test_fiber函数中resume函数下面统计指向fiber的智能指针数目，其实可以理解为本身加上GetThis函数数目，因为GetThis函数使用`shared_from_this()`函数返回了一个智能指针副本。

```c++
...
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

好乱啊，再梳理一下调用的顺序吧。

在test_fiber()函数中：

**fiber构造子协程**(传入回调函数run_in_fiber()，指明了子协程之后的入口函数为MainFunc)=>**resume()函数**=>内部的swapcontext()函数，跳转到MainFunc()=>内部执行回调函数，即之前传入的run_in_fiber()=>执行函数中的yield()函数=>通过内部的回到swapcontext()函数resume()函数。

所以其实有些函数是没有执行完的。

然后执行下面一个resume()函数，这个有点像是和上面的互补：

resume()函数=>通过内部中的swapcontext()函数到了yield()函数的末尾，即上次从yield()函数过来的位置=>回到调用yield()函数的run_in_fiber()中yield函数的下一行=>回到执行回调函数m_cb()的MainFunc函数中m_cb()的下一行=>释放智能指针，执行yield函数=>翻转又回到了resume()函数过来的下一行。

然后就结束退出了。

之后就是执行`reset`函数。

```c++
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

这个其实和fiber()子协程构造函数很像。这里其实是复用其栈空间再创建一个新协程。

然后是下面一个循环，运行`fiber->resume();`这次的流程和上次其实是一致的，就是回调函数不一样，所以后面会有不一样的。这里的回调函数是`run_in_fiber2()`，这个函数内没有yield函数，所以就直接返回就好。返回MainFunc函数之后，继续完成下面的，然后运行yield函数切换回resume函数，然后退出就好。

需要注意的是，运行完yield函数之后，当前运行协程将会变为主协程。第二次在run_in_fiber中也是这样。所以在最后一次resume之后就也是主协程了。

但是有一个疑问是最后析构函数删除的子协程，主协程似乎没有进行处理。但是在第二个线程好像删除子协程了mmm，明天再看看好了。
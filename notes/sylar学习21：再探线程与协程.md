# sylar21：再探线程与协程

关于`test_thread`中的样例，主体其实就是子线程的构造与销毁。

```c++
for(int i = 0; i < 3; i++) {
        // 带参数的线程用std::bind进行参数绑定
        sylar::Thread::ptr thr(new sylar::Thread(std::bind(func1, &arg), "thread_" + std::to_string(i)));
        thrs.push_back(thr);
    }

    for(int i = 0; i < 3; i++) {
        thrs[i]->join();
    }
```

子线程的构造主要就是这个函数：

```c++
int rt = pthread_create(&m_thread, nullptr, &Thread::run, this);
```

其中，run为创建好子线程之后的入口函数，即子线程真正要做的事情。

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
    // 在出构造函数之前，确保线程先跑起来，保证能够初始化id?
    //这里感觉有点问题，应该是让主线程继续动才对，而非博客中所说的。
    thread->m_semaphore.notify();
    cb();
    return 0;
}
```

子线程创建其实就是copy了一下父线程，所以一开始什么名字啊，id都不是自己的，需要再自己赋值一下。

这里涉及到一个有点争议的问题，就是关于代码：

```c++
thread->m_semaphore.notify();
```

通过调试发现，因为是信号量值加1，相当于让一个阻塞的线程重新运行。这里其实是让主线程继续运行。因为之后子线程要运行自己的回调函数，即cb()，之后其实主线程和子线程不会相互干扰的，所以是可以一起运行的。

在主线程中：

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

顺序相当于是先创建子线程，然后子线程进入run函数时(或前或后)，主线程运行到`m_semaphore.wait();`，然后就开始等待。可能是因为子线程在修改名字，创建阶段。然后子线程运行`thread->m_semaphore.notify();`接触主线程的等待，然后子线程运行cb()，主线程开始下一轮构造子线程。

然后还有一个点，就是在cb()中，也是在func1中有

```c++
for(int i = 0; i < 10000; i++) {
        sylar::Mutex::Lock lock(s_mutex);
        ++count;
    }
```

这里应该是防止线程与线程之间相互的影响。

运行的过程中可以发现，当退出当前循环时，会自动析构锁，即解锁，然后在下一个循环加锁这样。



然后是测试协程，即test_fiber.cc。

协程这里的样例和进程是相似的：

```c++
std::vector<sylar::Thread::ptr> thrs;
    for (int i = 0; i < 2; i++) {
        thrs.push_back(sylar::Thread::ptr(
            new sylar::Thread(&test_fiber, "thread_" + std::to_string(i))));
    }

    for (auto i : thrs) {
        i->join();
    }
```

大体上还是循环建立线程，然后在线程里面建立协程这样。

关于线程的逻辑就不再阐述了。有很多是不变的，例如线程生成函数，以及run函数，每个线程实际上要做的是线程的回调函数，根据回调函数任务的不同，线程可以实现不同的功能。

另外关于调试线程相关的测试函数，只需要在run函数中`thread->m_semaphore.notify();`之前打断点，然后开启`set scheduler-locking on`即可，这样主线程也不会过多的进行运行，不至于跑的贼快又生成了一个线程。

然后我们进入到cb()中打断点运行到此即可。

然后就是运行`sylar::Fiber::GetThis();`

这里这个函数是生成主线程的。

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

在这里的没有参数的fiber构造函数是生成主协程：

```C++
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

然后就是再生成一个子协程。

```C++

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

这个就是子协程的构造了。其实和主协程还是有很多不一样的地方的。比如关于栈的一些东西，主协程是没有的，可见主协程也不怎么干活，只是协调一下。因为刚进入线程其实也就是主协程中了，然后有的子协程才是真正要干活的。

当然了子协程内部有很多代码都是用来初始化的，然后主要用来干活的还是类似于run那样的入口函数，这里即时`MainFunc`。

另外两种协程有一个共同点就是`getcontext`函数，这个是为了保存当前协程的上下文。因为协程不是并行运行的，不想线程那样，我们一起在运行，协程则是一个运行完，下一个再运行，所以需要保存上下文，你运行完了，然后进行上下文的处置，然后我再运行。关于主协程和子协程的切换，其实这里设计的很巧妙。

首先，子协程运行`fiber->resume();`，即将fiber和当前运行的协程(主协程)进行切换。

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

这里就是进行切换协程，上面的SetThis函数是把当前协程表示换为子协程，然后把子协程的运行状态变为RUNNING。

下面是进行上下文的互换，即将主协程的上下文和当前子协程的上下文进行互换。互换之后就到了子协程里面了，就到了入口函数MainFunc中了。

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

首先就是获得当前协程，得知道当前子协程是哪个，这就是设置全局变量t_fiber的原因。

之后就是知行任务了，还是熟悉的回调函数。

```c++
void run_in_fiber() {
    SYLAR_LOG_INFO(g_logger) << "run_in_fiber begin";

    SYLAR_LOG_INFO(g_logger) << "before run_in_fiber yield";
    sylar::Fiber::GetThis()->yield();
    SYLAR_LOG_INFO(g_logger) << "after run_in_fiber yield";

    SYLAR_LOG_INFO(g_logger) << "run_in_fiber end";
    // fiber结束之后会自动返回主协程运行
}
```

回调函数中真正有用的还是`sylar::Fiber::GetThis()->yield();`。

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

不过看yield函数内部，其实就是再切换回去，其实也没干什么。总的来说就是这个回调函数其实没干什么。这里是将当前的上下文换为了主协程的上下文。然后就回到了主协程上次的运行位置。运行位置就是resume函数交换完的位置，即164行。

![image-20240712201640659](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240712201640659.png)

然后再退出调用resume函数的位置。

然后下面关于引用智能指针的计数还挺有意思的：

```C++
/** 
     * 关于fiber智能指针的引用计数为3的说明：
     * 一份在当前函数的fiber指针，一份在MainFunc的cur指针
     * 还有一份在在run_in_fiber的GetThis()结果的临时变量里
     */
    SYLAR_LOG_INFO(g_logger) << "use_count:" << fiber.use_count(); // 3

    SYLAR_LOG_INFO(g_logger) << "fiber status: " << fiber->getState(); // READY
```

我们知道如果运行结束了，智能指针就会自动销毁了，这里留下来的是还没有结束的子协程。

其实思考上面的过程，我们会发现子协程确实有很多函数只运行了一般，就没有再继续了。比如在入口函数MainFunc中只是调用了回调函数，后面还没有运行呢，比如yield切换回去了，但还没有把这个函数走完，然后返回呢，加上一开始这个协程的本身，所以这里引用计数为3。

然后我们再运行resume()函数，和主协程进行交换，进入子协程。

![image-20240712202357038](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240712202357038.png)

可以发现我们进入了上次子线程退出的位置。

然后返回回调函数run_in_fiber中，然后返回MainFunc中，然后运行到最后一行的yield，再次进行切换。

```c++
auto raw_ptr = cur.get(); // 手动让t_fiber的引用计数减1
cur.reset();
raw_ptr->yield();
```

不过这之后的fiber.use_count()就变为1了。因为在yield中的跟着退出函数而寄了，在MainFunc中的有手动让他寄了，所以最终就只剩下本身那个智能指针了。

```c++
fiber->reset(run_in_fiber2); // 上一个协程结束之后，复用其栈空间再创建一个新协程
fiber->resume();
```

然后这里就复用栈空间再创建了一个新协程。

然后子协程就像新的一样，重新运行了，不同的是这次进入的是新的回调函数，即run_in_fiber2。

这次不需要善后是因为在回调函数没有运行yield函数，所以子协程在MainFunc函数运行完之后就调用yield函数回去了。

ok感觉这里关于线程和协程的内容自己已经完全搞懂了。

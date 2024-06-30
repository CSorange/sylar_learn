# sylar学习07：定时器模块

首先是类**Timer**

```c++
/// 是否循环定时器
bool m_recurring = false;
/// 执行周期
uint64_t m_ms = 0;
/// 精确的执行时间
uint64_t m_next = 0;
/// 回调函数
std::function<void()> m_cb;
/// 定时器管理器
TimerManager* m_manager = nullptr;
```

operator()（仿函数）

```c++
bool Timer::Comparator::operator()(const Timer::ptr& lhs
                        ,const Timer::ptr& rhs) const {
    if(!lhs && !rhs) {
        return false;
    }
    if(!lhs) {
        return true;
    }
    if(!rhs) {
        return false;
    }
    if(lhs->m_next < rhs->m_next) {
        return true;
    }
    if(rhs->m_next < lhs->m_next) {
        return false;
    }
    return lhs.get() < rhs.get();
}
```

这里是比较定时器的智能指针的大小(按执行时间排序)。

可以看到，首先查看有无精准执行时间为0的，之后比较两者的执行时间，如果执行时间相等，则比较智能指针的地址。

这里比较大小就是为了可以放到时间管理器的set中。

构造函数：

```c++
Timer::Timer(uint64_t ms, std::function<void()> cb,
             bool recurring, TimerManager* manager)
    :m_recurring(recurring)
    ,m_ms(ms)
    ,m_cb(cb)
    ,m_manager(manager) {
    // 执行时间为当前时间+执行周期
    m_next = sylar::GetElapsedMS() + m_ms;
}
```

由`TimerManager`创建`Timer`

cancel（取消定时器）

```c++
bool Timer::cancel() {
    TimerManager::RWMutexType::WriteLock lock(m_manager->m_mutex);
    if(m_cb) {
        m_cb = nullptr;
        auto it = m_manager->m_timers.find(shared_from_this());// 在set中找到自身定时器
        m_manager->m_timers.erase(it);// 找到删除
        return true;
    }
    return false;
}
```

这里其实没有很理解，相当于是TimerManager是总领所有的，然后当前线程/协程找到自身的定时器然后再删除这样么，shared_from_this()函数应该是不会新建一个定时器的。

refresh（刷新执行时间）

```c++
bool Timer::refresh() {
    TimerManager::RWMutexType::WriteLock lock(m_manager->m_mutex);
    if(!m_cb) {
        return false;
    }
    auto it = m_manager->m_timers.find(shared_from_this());// 在set中找到自身定时器
    if(it == m_manager->m_timers.end()) {
        return false;
    }
    m_manager->m_timers.erase(it);// 删除
    m_next = sylar::GetElapsedMS() + m_ms;// 更新执行时间
    m_manager->m_timers.insert(shared_from_this());// 重新插入，这样能按最新的时间排序
    return true;
}
```

这个其实很好理解，更新其实就是找到并删除老的定时器，然后再将新的添加进去。

reset（重置定时器时间）

```c++
bool Timer::reset(uint64_t ms, bool from_now) {//from_now表示是否从当前时间开始计算
    if(ms == m_ms && !from_now) {
        return true;
    }
    TimerManager::RWMutexType::WriteLock lock(m_manager->m_mutex);
    if(!m_cb) {
        return false;
    }
    auto it = m_manager->m_timers.find(shared_from_this());// 在set中找到自身定时器
    if(it == m_manager->m_timers.end()) {
        return false;
    }
    m_manager->m_timers.erase(it);// 删除定时器
    uint64_t start = 0; // 起始时间
    if(from_now) {// 从现在开始计算
        start = sylar::GetElapsedMS();
    } else {
        start = m_next - m_ms;
    }
    m_ms = ms;
    m_next = start + m_ms;
    m_manager->addTimer(shared_from_this(), lock);
    return true;

}
```

这里自己不是很懂mmm，可能是执行周期的概念不是很了解。

下面是关于类TimerManager

```c++
/// Mutex
RWMutexType m_mutex;
/// 定时器集合
std::set<Timer::ptr, Timer::Comparator> m_timers;
/// 是否触发onTimerInsertedAtFront
bool m_tickled = false;
/// 上次执行时间
uint64_t m_previouseTime = 0;
```

所以其实上面的Timer是一个定时器，然后存到了TimerManager中的定时器集合里面。

构造函数：

```c++
TimerManager::TimerManager() {
    m_previouseTime = sylar::GetElapsedMS();
}
```

仅仅是记录下当下的时间。

addTimer（添加定时器）

```c++
Timer::ptr TimerManager::addTimer(uint64_t ms, std::function<void()> cb
                                  ,bool recurring) {
    Timer::ptr timer(new Timer(ms, cb, recurring, this));// 创建定时器
    RWMutexType::WriteLock lock(m_mutex);
    addTimer(timer, lock);// 添加到set中
    return timer;
}
void TimerManager::addTimer(Timer::ptr val, RWMutexType::WriteLock& lock) {
    auto it = m_timers.insert(val).first;// 添加到set中并且拿到迭代器
    bool at_front = (it == m_timers.begin()) && !m_tickled;// 如果该定时器是超时时间最短 并且 没有设置触发onTimerInsertedAtFront
    if(at_front) {// 设置触发onTimerInsertedAtFront
        m_tickled = true;
    }
    lock.unlock();

    if(at_front) {
        onTimerInsertedAtFront();//就是在IOManager中就是做了一次tickle()的操作
    }
}
```

这里就是创建了一个定时器并放入到set中去。可以看到创建定时器时也需要记录定时器管理类，就用一个this指针来进行传递。

这里判断是否是超时最短，不太理解，为什么要判断这个。

addConditionTimer（添加条件定时器）

```c++
Timer::ptr TimerManager::addConditionTimer(uint64_t ms, std::function<void()> cb,std::weak_ptr<void> weak_cond,bool recurring) {
    return addTimer(ms, std::bind(&OnTimer, weak_cond, cb), recurring);
    // 在定时器触发时会调用 OnTimer 函数，并在OnTimer函数中判断条件对象是否存在，如果存在则调用回调函数cb。
}
static void OnTimer(std::weak_ptr<void> weak_cond, std::function<void()> cb) {
    std::shared_ptr<void> tmp = weak_cond.lock();// 它首先使用weak_cond的lock函数获取一个shared_ptr指针tmp
    if(tmp) {// 如果tmp不为空，则调用回调函数cb。
        cb();
    }
}
```

`weak_ptr`是一种弱引用，它不会增加所指对象的引用计数，也不会阻止所指对象被销毁。

而`shared_ptr`是一种强引用，它会增加所指对象的引用计数，直到所有`shared_ptr`都被销毁才会释放所指对象的内存。

在这段代码中，`weak_cond`是一个`weak_ptr`类型的对象，通过调用它的`lock()`方法可以得到一个`shared_ptr`类型的对象`tmp`，如果`weak_ptr`已经失效，则`lock()`方法返回一个空的`shared_ptr`对象。

确实相比addTimer多绕了一下，在OnTimer函数中通过一个weak_ptr获得shared_ptr，如果有才进行之后的cb()，相当于回调函数中有回调函数。

getNextTimer（ 最近一个定时器执行的时间间隔）

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

这个函数让自己去思考定时器的意义。我感觉应该是set中有一系列的定时器，每个定时器都记录着本定时器到什么时候可以启动。这个函数就是获取最快可以启动的那个定时器，还有多场时间启动，或者这个定时器已经超时了。

这个思想和禁忌表有点像，不过禁忌表还是太朴素了mmm。

listExpiredCb（定时器执行列表）

```c++
void TimerManager::listExpiredCb(std::vector<std::function<void()> >& cbs) {
    uint64_t now_ms = sylar::GetElapsedMS();
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
    bool rollover = false;
    if(SYLAR_UNLIKELY(detectClockRollover(now_ms))) {
        // 使用clock_gettime(CLOCK_MONOTONIC_RAW)，应该不可能出现时间回退的问题
        rollover = true;
    }
    if(!rollover && ((*m_timers.begin())->m_next > now_ms)) {
        return;
    } // 如果服务器时间没问题，并且第一个定时器都没有到执行时间，就说明没有任务需要执行
    Timer::ptr now_timer(new Timer(now_ms));
    //如果时间改动，则所有Timer被视为过期，如果没有，则返回比当前时间晚完成的定时器，早完成的都已经过期了。
    auto it = rollover ? m_timers.end() : m_timers.lower_bound(now_timer);
    while(it != m_timers.end() && (*it)->m_next == now_ms) {// 筛选出当前时间等于next时间执行的Timer
        ++it;
    }
    expired.insert(expired.begin(), m_timers.begin(), it);// 将这些Timer都加入到expired中
    m_timers.erase(m_timers.begin(), it);// 将已经放入expired的定时器删掉
    cbs.reserve(expired.size());

    for(auto& timer : expired) {
        cbs.push_back(timer->m_cb);// 将expired的timer放入到cbs中
        if(timer->m_recurring) {// 如果是循环定时器，则再次放入定时器集合中
            timer->m_next = now_ms + timer->m_ms;
            m_timers.insert(timer);
        } else {
            timer->m_cb = nullptr;
        }
    }
}

bool TimerManager::detectClockRollover(uint64_t now_ms) {
    bool rollover = false;
    if(now_ms < m_previouseTime &&
            now_ms < (m_previouseTime - 60 * 60 * 1000)) {
        rollover = true;
    }// 如果当前时间比上次执行时间还小 并且 小于一个小时的时间，相当于时间倒流了
    m_previouseTime = now_ms;
    return rollover;
}
```

这个部分自己可以看懂，思想还是很朴素的hh。 另外其实这里的时间单位都是微秒，所以可以看到很多*1000，是需要先转换成秒。

hasTimer（是否有定时器）

```c++
bool TimerManager::hasTimer() {
    RWMutexType::ReadLock lock(m_mutex);
    return !m_timers.empty();
}
```



之后我们看`test_timer.cc`测试文件。可以看到，除了常规的读取命令行以及加载配置文件，剩下关于定时器的就是函数`test_timer`。

```c++
void test_timer() {
    sylar::IOManager iom;

    // 循环定时器
    s_timer = iom.addTimer(1000, timer_callback, true);
    
    // 单次定时器
    iom.addTimer(500, []{
        SYLAR_LOG_INFO(g_logger) << "500ms timeout";
    });
    iom.addTimer(5000, []{
        SYLAR_LOG_INFO(g_logger) << "5000ms timeout";
    });
}
```

这里用到了IO协程调度，被自己跳过了mmm。如果单看类IOManger，可以看到是继承Scheduler和TimerManager类的，感觉是这两个的组合。运行addTimer函数。

```C++
s_timer = iom.addTimer(1000, timer_callback, true);
Timer::ptr TimerManager::addTimer(uint64_t ms, std::function<void()> cb,bool recurring) {
    Timer::ptr timer(new Timer(ms, cb, recurring, this));// 创建定时器
    RWMutexType::WriteLock lock(m_mutex);
    addTimer(timer, lock);// 添加到set中
    return timer;
}
```

其中回调函数是timer_callback，执行周期为1s，开启循环定时器。我们创建一个新的定时器并把它添加到定时器集合中。

如果循环定时器到时了，会进行回调函数中的操作：

```c++
void timer_callback() {
    SYLAR_LOG_INFO(g_logger) << "timer callback, timeout = " << timeout;
    timeout += 1000;
    if(timeout < 5000) {
        s_timer->reset(timeout, true);
    } else {
        s_timer->cancel();
    }
}
```

可以看到，如果到5000ms，则退出，如果没有，则重新加入。

其实可以看到，这些都还蛮好理解的。但应该有一个线程始终在检测定时器，但在这个函数代码中其实并没有看到，有理由怀疑是在IO中进行的。

所以先进入IO中看一下吧，可以看到其实IO就是在Scheduler和TimerManager类基础上进行封装的。

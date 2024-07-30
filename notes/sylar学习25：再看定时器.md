# sylar25：再看定时器

本来自己想把定时器跳过的，毕竟没什么营养，但自己不清楚定时器到时间了之后是如何通知，所以还是再看一下吧。

> `IOManager`继承`TimerManager`，在`idle`中与`epoll_wait`的超时时间相结合，毫秒级的精度，在指定的超时时间结束后执行回调函数。

与epoll_wait相结合么？？

定时器是采用堆的方式进行组织的。因为我们会先处理超时时间最小的定时器，然后再处理其他的，所以和最小堆的特性最为契合。超时时间到后，获取当前的绝对时间，并且把时间堆中已经超时的所有定时器都收到一个容器中，执行他们的回调函数。

定时器的时间都是用绝对时间来计算的，和禁忌表中使用绝对时间的原理是一样的。

>在`IOmanager`中，将定时器与`epoll_wait`的超时时间相结合，等待时间为定时器中超时时间最小的时间。若是超时导致`epoll_wait`返回，那么一定是定时器设置的超时时间到了，把已经超时的定时器的任务加入到任务队列中去执行，但也有可能是别的信号使`epoll_wait`返回，那么仍然需要判断是否有定时器超时，所以要比较当前的绝对时间与定时器的绝对时间来判断有无定时器超时。

即epoll_wait返回有两种情况，一种是定时器设置的超时时间到了，另一种是别的信号，比如要IO操作了这些，所以返回之后我们首先比较定时器时间和当前时间。

首先看Timer构造函数，只能被TimerManager所创建(有参数TimerManager)。

```c++
Timer::Timer(uint64_t ms, std::function<void()> cb,
             bool recurring, TimerManager* manager)
    :m_recurring(recurring)
    ,m_ms(ms)
    ,m_cb(cb)
    ,m_manager(manager) {
    m_next = sylar::GetElapsedMS() + m_ms;
}
```

**cancel**取消定时器，就是在set中找到，然后删除。

这里其实有一个问题，在看成员变量的时候发现定时器集合是用set来储存的。`std::set<Timer::ptr, Timer::Comparator> m_timers;`这和上面说的是用堆有点出入。

**refresh**刷新执行时间，首先找到自身定时器，然后删除，然后更新执行时间之后重新插入。这里更新之后的执行时间为`m_next = sylar::GetElapsedMS() + m_ms;`自己不是很明白mmm。

**reset**重置定时器时间，也不是很懂mmm，主要成员变量中m_ms(执行周期)和m_next(精确的执行时间)自己不是很明白什么意思。

这些函数都是从自身定时器本身出发的，下面的TimerManager是从一个更加宏观的角度来看所有的定时器的。

**addTimer**即添加一个定时器。

**addConditionTimer**为添加条件定时器。

```c++
static void OnTimer(std::weak_ptr<void> weak_cond, std::function<void()> cb) {
    std::shared_ptr<void> tmp = weak_cond.lock();
    if(tmp) {
        cb();
    }
}

Timer::ptr TimerManager::addConditionTimer(uint64_t ms, std::function<void()> cb,std::weak_ptr<void> weak_cond,bool recurring) {
    return addTimer(ms, std::bind(&OnTimer, weak_cond, cb), recurring);
}
```

其中OnTimer为回调函数。这里用到了一个弱引用weak_ptr。弱引用不会增加所指对象的引用计数，也不会阻止所指对象被销毁。

我们调用lock()得到了一个shared_ptr类型的对象，如果weak_ptr失效了，则lock()返回一个空的share_ptr对象，然后就执行不了回调函数了。

这里其实就是用的弱引用的特点，在do_io中我们可以看到这个函数的应用。

**getNextTimer**获得最近一个定时器执行的时间间隔，如果最近定时器超时了，就表示该执行了，返回0。如果还没有到时间，则返回还要多久才可以执行。

**listExpiredCb**为获取要执行的定时器列表。这里比较当前时间和定时器集合中的时间，如果定时器的时间早于当前时间，则表示这个定时器应该执行了，然后就把这个定时器对应的cb回调函数加入需要运行的cb数组中，把这些定时器全部移除。如果其中有循环定时器，就再加入进去。

所以成员变量m_recurring和m_ms应该是用来处理循环定时器的hh。m_next才是精确的执行时间，即每次我们要和当前时间进行比较的量。



结合完这里再看idle我就明白了，关于定时器其实是在idle里处理的，只是自己上次看的时候跳过了。

```C++
void IOManager::idle() {
    SYLAR_LOG_DEBUG(g_logger) << "idle";

    // 一次epoll_wait最多检测256个就绪事件，如果就绪事件超过了这个数，那么会在下轮epoll_wati继续处理
    const uint64_t MAX_EVNETS = 256;
    epoll_event *events       = new epoll_event[MAX_EVNETS]();
    std::shared_ptr<epoll_event> shared_events(events, [](epoll_event *ptr) {
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
            }
            rt = epoll_wait(m_epfd, events, MAX_EVNETS, (int)next_timeout);
            if(rt < 0 && errno == EINTR) {
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
                schedule(cb);
            }
            cbs.clear();
        }
        
        // 遍历所有发生的事件，根据epoll_event的私有指针找到对应的FdContext，进行事件处理
        for (int i = 0; i < rt; ++i) {
            epoll_event &event = events[i];
            if (event.data.fd == m_tickleFds[0]) {
                // ticklefd[0]用于通知协程调度，这时只需要把管道里的内容读完即可
                uint8_t dummy[256];
                while (read(m_tickleFds[0], dummy, sizeof(dummy)) > 0)
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
            if (event.events & EPOLLIN) {
                real_events |= READ;
            }
            if (event.events & EPOLLOUT) {
                real_events |= WRITE;
            }

            if ((fd_ctx->events & real_events) == NONE) {
                continue;
            }

            // 剔除已经发生的事件，将剩下的事件重新加入epoll_wait
            int left_events = (fd_ctx->events & ~real_events);
            int op          = left_events ? EPOLL_CTL_MOD : EPOLL_CTL_DEL;
            event.events    = EPOLLET | left_events;

            int rt2 = epoll_ctl(m_epfd, op, fd_ctx->fd, &event);
            if (rt2) {
                SYLAR_LOG_ERROR(g_logger) << "epoll_ctl(" << m_epfd << ", "
                                          << (EpollCtlOp)op << ", " << fd_ctx->fd << ", " << (EPOLL_EVENTS)event.events << "):"
                                          << rt2 << " (" << errno << ") (" << strerror(errno) << ")";
                continue;
            }

            // 处理已经发生的事件，也就是让调度器调度指定的函数或协程
            if (real_events & READ) {
                fd_ctx->triggerEvent(READ);
                --m_pendingEventCount;
            }
            if (real_events & WRITE) {
                fd_ctx->triggerEvent(WRITE);
                --m_pendingEventCount;
            }
        } // end for

        /**
         * 一旦处理完所有的事件，idle协程yield，这样可以让调度协程(Scheduler::run)重新检查是否有新任务要调度
         * 上面triggerEvent实际也只是把对应的fiber重新加入调度，要执行的话还要等idle协程退出
         */ 
        Fiber::ptr cur = Fiber::GetThis();
        auto raw_ptr   = cur.get();
        cur.reset();

        raw_ptr->yield();
    } // end while(true)
}
```

首先我们获取了第一个定时器是否超时或者还有多久才到时间：

```c++
uint64_t next_timeout = 0;
        if( SYLAR_UNLIKELY(stopping(next_timeout))) {
            SYLAR_LOG_DEBUG(g_logger) << "name=" << getName() << "idle stopping exit";
            break;
        }

bool IOManager::stopping(uint64_t &timeout) {
    // 对于IOManager而言，必须等所有待调度的IO事件都执行完了才可以退出
    // 增加定时器功能后，还应该保证没有剩余的定时器待触发
    timeout = getNextTimer();
    return timeout == ~0ull && m_pendingEventCount == 0 && Scheduler::stopping();
}
```

这里的next_timeout就是，在stopping中其实执行了getNextTimer()，获取了第一个定时器是否超时或者还有多久才到时间。

```c++
do{
            // 默认超时时间5秒，如果下一个定时器的超时时间大于5秒，仍以5秒来计算超时，避免定时器超时时间太大时，epoll_wait一直阻塞
            static const int MAX_TIMEOUT = 5000;
            if(next_timeout != ~0ull) {
                next_timeout = std::min((int)next_timeout, MAX_TIMEOUT);
            } else {
                next_timeout = MAX_TIMEOUT;
            }
            rt = epoll_wait(m_epfd, events, MAX_EVNETS, (int)next_timeout);
            if(rt < 0 && errno == EINTR) {
                continue;
            } else {
                break;
            }
        } while(true);
```

在循环里面可以看到，epoll_wait中使用了参数next_timeout，即观察到没有事件发生最多等待的时间。如果这个值为0，则表示不进行等待。

在之后的跳出条件中，只有rt < 0 && errno == EINTR的条件下才会继续，即中断发生的时候，一般是都会跳出的。

之后进行了两大操作，其实这分别对应着对定时器的操作与对fd文件的操作。

```c++
// 收集所有已超时的定时器，执行回调函数
        std::vector<std::function<void()>> cbs;
        listExpiredCb(cbs);
        if(!cbs.empty()) {
            for(const auto &cb : cbs) {
                schedule(cb);
            }
            cbs.clear();
        }
```

这是对定时器中的回调函数的操作。即收集所有到时的定时器，然后将这些定时器的回调函数加入schedule中进行运行。

```c++
for (int i = 0; i < rt; ++i) {
            epoll_event &event = events[i];
            if (event.data.fd == m_tickleFds[0]) {
                // ticklefd[0]用于通知协程调度，这时只需要把管道里的内容读完即可
                uint8_t dummy[256];
                while (read(m_tickleFds[0], dummy, sizeof(dummy)) > 0)
                    ;
                continue;
            }
    ...
}
```

这是对epoll_wait返回值的操作。这里又分为两种情况，一种是通过通道过来了数据，其实自己不太明白是如何通过通道来的。而且在调试的过程中也发现了会传输具体的数据的。

![image-20240719173810885](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240719173810885.png)

在测试中有两次通道有传输的数据(即rt为1的情况)，输出的都是"TT"，自己也不清楚为什么是这个。

另一种情况就是通过套接字socket传输的数据，当然在test_timer中没有。

所以其实这两种情况是并列的，定时器与fd。这下自己明白了。


# sylar学习24：再看hook

hook模块没有对应的test文件，但在很多地方都可以看到他的身影。

> `hook`模块封装了一些C标准库提供的`API`，`socket IO`相关的`API`。能够使同步`API`实现异步的性能。

这次学习的主要目的就是搞清楚是如何使同步API实现异步的性能的。

其实关于封装自己是清楚的，相当于是在使用原接口的基础上还做了一些自己想做的事情，然后还用原接口的名字命名自己做的事情以及调用的接口。

而且hook操作只针对socket文件，其他文件执行原本的API。

关于同步API实现异步操作的意思，这里举一个例子。

>如果没有hook：要先等睡3秒，然后再发送，等发送完再接收数据，一旦有个任务导致协程阻塞，整个线程就阻塞掉了。(相当于是一个接着一个进行)
>
>如果hook了：
>
>1.那么在执行`sleep`时就先添加个3秒的定时器，回调函数为继续调度本协程，然后让协程让出执行权。
>
>2.可以紧接着执行`write`操作，不用再等待3秒钟了，当调度器执行`write`会先注册一个写事件，回调函数是让当前协程唤醒继续执行，然后让协程让出执行权。
>
>3.执行完`write`可以紧接着执行`read`，在调度器上注册一个读事件，回调函数是当让当前协程唤醒继续执行，让后让协程让出执行权。
>
>4.等3秒后，执行定时器的回调函数，让协程继续调用sleep。
>
>5.在等待3秒的过程中，一旦可写，那么就继续执行写的协程。
>
>6.一旦可读，那么就继续执行读的协程。

如何同步API实现异步功能，主要是在函数do_io中。

```c++
template<typename OriginFun, typename... Args>
static ssize_t do_io(int fd, OriginFun fun, const char* hook_fun_name,
        uint32_t event, int timeout_so, Args&&... args) {
    if(!sylar::t_hook_enable) {// 如果不hook，直接返回原接口
        return fun(fd, std::forward<Args>(args)...); //可以看到，fun其实就是原本要运行的函数
    }

    sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(fd);// 获取fd对应的FdCtx
    if(!ctx) {
        return fun(fd, std::forward<Args>(args)...);
    }

    if(ctx->isClose()) {// 看看句柄是否关闭
        errno = EBADF;
        return -1;
    }

    if(!ctx->isSocket() || ctx->getUserNonblock()) {// 不是socket 或 用户设置了非阻塞
        return fun(fd, std::forward<Args>(args)...);
    }
    //下面就是要进行异步的操作。
    uint64_t to = ctx->getTimeout(timeout_so);// 不是socket 或 用户设置了非阻塞
    std::shared_ptr<timer_info> tinfo(new timer_info);// 设置超时条件

retry:
    ssize_t n = fun(fd, std::forward<Args>(args)...);// 先执行fun 读数据或写数据 若函数返回值有效就直接返回
    while(n == -1 && errno == EINTR) {// 若中断则重试
        n = fun(fd, std::forward<Args>(args)...);
    }
    if(n == -1 && errno == EAGAIN) {// 若为阻塞状态
        sylar::IOManager* iom = sylar::IOManager::GetThis();// 获得当前IO调度器
        sylar::Timer::ptr timer;// 定时器
        std::weak_ptr<timer_info> winfo(tinfo);

        if(to != (uint64_t)-1) {// 说明设置了超时时间
            timer = iom->addConditionTimer(to, [winfo, fd, iom, event]() {//添加条件定时器，to时间消息还没来就触发callback
                auto t = winfo.lock(); //这里是回调函数部分
                if(!t || t->cancelled) {//tinfo失效 || 设了错误
                    return;
                }
                t->cancelled = ETIMEDOUT;// 没错误的话设置为超时而失败
                iom->cancelEvent(fd, (sylar::IOManager::Event)(event));//取消事件，触发回调
            }, winfo);
        }

        int rt = iom->addEvent(fd, (sylar::IOManager::Event)(event));//cb为空， 任务为执行当前协程
        if(SYLAR_UNLIKELY(rt)) {// addEvent失败， 取消上面加的定时器
            SYLAR_LOG_ERROR(g_logger) << hook_fun_name << " addEvent("
                << fd << ", " << event << ")";
            if(timer) {
                timer->cancel();
            }
            return -1;
        } else {
            sylar::Fiber::GetThis()->yield();//然后就切换协程了，回来可能因为：1.超时了， timer cancelEvent triggerEvent会唤醒回来2.addEvent数据回来了会唤醒回来
            if(timer) {// 回来了还有定时器就取消掉
                timer->cancel();
            }
            if(tinfo->cancelled) {// 从定时任务唤醒，超时失败
                errno = tinfo->cancelled;
                return -1;
            }
            goto retry;
        }
    }
    
    return n;
}
```

这个其实可以结合上面的那个样例来进行理解。

1.先进行判断，是否按原函数进行执行。

```c++
if(!sylar::t_hook_enable) {// 如果不hook，直接返回原接口
        return fun(fd, std::forward<Args>(args)...); //可以看到，fun其实就是原本要运行的函数
    }

    sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(fd);// 获取fd对应的FdCtx
    if(!ctx) {
        return fun(fd, std::forward<Args>(args)...);
    }

    if(ctx->isClose()) {// 看看句柄是否关闭
        errno = EBADF;
        return -1;
    }

    if(!ctx->isSocket() || ctx->getUserNonblock()) {// 不是socket 或 用户设置了非阻塞
        return fun(fd, std::forward<Args>(args)...);
    }
```

然后是进行异步相关的操作：

首先我们获得超时时间to，在阻塞态之前(即`if(n == -1 && errno == EAGAIN)`)我们都是非阻塞的，即不断尝试运行fun函数。

如果是阻塞态，我们设置超时时间，即现在不是在阻塞嘛，但我也不能等太久，如果超时时间到了还没有动静，我就要执行回调函数了。

这个定时器相当的长，主要是cb直接写了函数体。

```c++
timer = iom->addConditionTimer(to, [winfo, fd, iom, event]() {//添加条件定时器，to时间消息还没来就触发callback
     auto t = winfo.lock(); //这里是回调函数部分
     if(!t || t->cancelled) {//tinfo失效 || 设了错误
         return;
     }
     t->cancelled = ETIMEDOUT;// 没错误的话设置为超时而失败
     iom->cancelEvent(fd, (sylar::IOManager::Event)(event));//取消事件，触发回调
            }, winfo);
```

其实中间部分都是一个函数体。

如果没动静执行了回调函数，回调函数里有什么呢，就是看看什么个情况。如果有错误，就直接返回就好了，但如果没有错误，那就是超时了，就把这个event用cancelEvent强行删除。

如果超时时间没有到，数据就来了，执行完do_io后，弱指针就是个空指针，就不会执行定时器的回调函数了。

然后继续执行

```c++
int rt = iom->addEvent(fd, (sylar::IOManager::Event)(event));//cb为空， 任务为执行当前协程
```

首先添加event，这里cb为空，就相当于是回到当前协程。

然后就看是否添加成功了。如果失败了：

```c++
if(SYLAR_UNLIKELY(rt)) {// addEvent失败， 取消上面加的定时器
            SYLAR_LOG_ERROR(g_logger) << hook_fun_name << " addEvent("
                << fd << ", " << event << ")";
            if(timer) {
                timer->cancel();
            }
            return -1;
        } 
```

将定时器删除并且返回错误。

如果成功了：

```c++
else {
            sylar::Fiber::GetThis()->yield();//然后就切换协程了，回来可能因为：1.超时了， timer cancelEvent triggerEvent会唤醒回来2.addEvent数据回来了会唤醒回来
            if(timer) {// 回来了还有定时器就取消掉
                timer->cancel();
            }
            if(tinfo->cancelled) {// 从定时任务唤醒，超时失败
                errno = tinfo->cancelled;
                return -1;
            }
            goto retry;
        }
```

然后就是熟悉的yield函数，切换协程，让其他没有阻塞的任务来进行。如果切换回来了，就是到yield下面的位置了。

什么情况会切换协程呢，可能因为超时了，或者数据来了。如果超时了就结束了，如果是因为数据来了，就回到retry的位置，就重新开始新的一轮了。

总的来说是和上面举的例子相对应的。

然后就是上面提到的sleep函数：

```c++
unsigned int sleep(unsigned int seconds) {
    if(!sylar::t_hook_enable) {
        return sleep_f(seconds);
    }

    sylar::Fiber::ptr fiber = sylar::Fiber::GetThis();
    sylar::IOManager* iom = sylar::IOManager::GetThis();
    iom->addTimer(seconds * 1000, std::bind((void(sylar::Scheduler::*)
            (sylar::Fiber::ptr, int thread))&sylar::IOManager::schedule
            ,iom, fiber, -1));
    sylar::Fiber::GetThis()->yield();
    return 0;
}
```

这个函数其实就做了两个事情，一个是添加定时器，另一个是使用yield让出执行权。

在添加定时器中的回调函数没有看懂mmm。直观上感受应该是Scheduler函数，即添加任务到m_tasks中去，但后面还有&还有很多东西，就不是很明白了。

之后是socket函数，这里是否使用hook其实差别没有很大。

```c++
int socket(int domain, int type, int protocol) {
    if(!sylar::t_hook_enable) {
        return socket_f(domain, type, protocol);
    }
    int fd = socket_f(domain, type, protocol);
    if(fd == -1) {
        return fd;
    }
    sylar::FdMgr::GetInstance()->get(fd, true);
    return fd;
}
```

然后是connect函数，这个其实和do_io差不多，你想do_io其实相当于是io操作，即读或者写，然后这里是连接，其实感觉和看的那个什么eator有些相似，读写和连接是并行且相似的操作。

```c++
int connect_with_timeout(int fd, const struct sockaddr* addr, socklen_t addrlen, uint64_t timeout_ms) {
    if(!sylar::t_hook_enable) {
        return connect_f(fd, addr, addrlen);
    }
    sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(fd);
    if(!ctx || ctx->isClose()) {
        errno = EBADF;
        return -1;
    }

    if(!ctx->isSocket()) {
        return connect_f(fd, addr, addrlen);
    }

    if(ctx->getUserNonblock()) {
        return connect_f(fd, addr, addrlen);
    }

    int n = connect_f(fd, addr, addrlen);
    if(n == 0) {
        return 0;
    } else if(n != -1 || errno != EINPROGRESS) {
        return n;
    }

    sylar::IOManager* iom = sylar::IOManager::GetThis();
    sylar::Timer::ptr timer;
    std::shared_ptr<timer_info> tinfo(new timer_info);
    std::weak_ptr<timer_info> winfo(tinfo);

    if(timeout_ms != (uint64_t)-1) {
        timer = iom->addConditionTimer(timeout_ms, [winfo, fd, iom]() {
                auto t = winfo.lock();
                if(!t || t->cancelled) {
                    return;
                }
                t->cancelled = ETIMEDOUT;
                iom->cancelEvent(fd, sylar::IOManager::WRITE);
        }, winfo);
    }

    int rt = iom->addEvent(fd, sylar::IOManager::WRITE);
    if(rt == 0) {
        sylar::Fiber::GetThis()->yield();
        if(timer) {
            timer->cancel();
        }
        if(tinfo->cancelled) {
            errno = tinfo->cancelled;
            return -1;
        }
    } else {
        if(timer) {
            timer->cancel();
        }
        SYLAR_LOG_ERROR(g_logger) << "connect addEvent(" << fd << ", WRITE) error";
    }

    int error = 0;
    socklen_t len = sizeof(int);
    if(-1 == getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len)) {
        return -1;
    }
    if(!error) {
        return 0;
    } else {
        errno = error;
        return -1;
    }
}
```

然后是accept函数：

```c++
int accept(int s, struct sockaddr *addr, socklen_t *addrlen) {
    int fd = do_io(s, accept_f, "accept", sylar::IOManager::READ, SO_RCVTIMEO, addr, addrlen);
    if(fd >= 0) {
        sylar::FdMgr::GetInstance()->get(fd, true);
    }
    return fd;
}
```

是通过do_io来实现的，然后加上了对文件的处理。

至于其他一些发送接收的操作，其实也是通过do_io来实现的。

最后是close，关闭socket。

```C++
int close(int fd) {
    if(!sylar::t_hook_enable) {
        return close_f(fd);
    }

    sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(fd);
    if(ctx) {
        auto iom = sylar::IOManager::GetThis();
        if(iom) {
            iom->cancelAll(fd);
        }
        sylar::FdMgr::GetInstance()->del(fd);
    }
    return close_f(fd);
}
```

同样的，如果不用hook直接进行就好了，如果使用的话，就多了一些对文件的处理。



定时器还是也看一下吧，自己不清楚是如何到时间提醒的，按说只有一个进程/线程，没有办法实现并行操作的mmm。

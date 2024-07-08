# sylar学习10：hook模块概述

hook模块概述

> `hook`模块封装了一些C标准库提供的`API`，`socket IO`相关的`API`。能够使同步`API`实现异步的性能。
>
> `hook`将`API`封装成一个与原始系统调用同名的接口，在调用这个接口时，先实现一些别的操作，然后在调用原始的系统`API`。这样对开发者来说很方便，不用重新学习新的接口，用着同步的接口实现异步的操作。
>
> 如果不使用`IOManager`，那么`hook`将没有作用，在`IOManager::run()`中使用`set_hook_enable()`将当前线程`hook`。
>
> 例如，当`IOManager`设置为一个线程执行时，通过`scheduler`添加了3个任务，`sleep(3)`，`write`，`read`。每个任务是在协程上按顺序执行的。
>
> ...
>
> 只对`socket`文件进行`hook`操作，其他的文件执行原本的`API`，为了更好的管理`socket`文件，使用`fdmanager`管理每个`socket fd`，并且为每个`socket`文件隐式的设置为`O_NONBLOCK`非阻塞，为了统一每一个`API`的统一性，所以也`hook`了`fcntl_f`等函数。

> `hook`的实现机制非常简单，就是通过动态库的全局符号介入功能，用自定义的接口来替换掉同名的系统调用接口。由于系统调用接口基本上是由C标准函数库`libc`提供的，所以这里要做的事情就是用自定义的动态库来覆盖掉`libc`中的同名符号。
>
> 由于动态库的全局符号介入问题，全局符号表只会记录第一次识别到的符号，后续的同名符号都被忽略，但这并不表示同名符号所在的动态库完全不会加载，因为有可能其他的符号会用到。以`libc`库举例，如果用户在链接`libc`库之前链接了一个指定的库，并且在这个库里实现了`read/write`接口，那么在程序运行时，程序调用的`read/write`接口就是指定库里的，而不是`libc`库里的。`libc`库仍然会被加载，因为`libc`库是程序的运行时库，程序不可能不依赖`libc`里的其他接口。因为`libc`库也被加载了，所以，通过一定的手段，仍然可以从`libc`中拿到属于`libc`的`read/write`接口，这就为`hook`创建了条件。程序可以定义自己的`read/write`接口，在接口内部先实现一些相关的操作，然后再调用`libc`里的`read/write`接口。

举个例子，相当于自己实现了一个myread()函数，然后在这个函数中调用标准库函数read()函数，而且在调用之前myread()函数中还实现了一些自己想实现的其他操作。

更重要的是想用同步的接口实现异步的操作，因为目前实现的myread函数中，都是判断一下，再进行read操作，推测在这里实现了异步的操作。

关于有无hook的区别：

> 例如，当`IOManager`设置为一个线程执行时，通过`scheduler`添加了3个任务，`sleep(3)`，`write`，`read`。每个任务是在协程上按顺序执行的。
>
> 如果没有`hook`：
>
> 要先等睡3秒，然后再发送，等发送完再接收数据，一旦有个任务导致协程阻塞，整个线程就阻塞掉了。
>
> 如果`hook`了：
>
> 那么在执行`sleep`时就先添加个3秒的定时器，回调函数为继续调度本协程，然后让协程让出执行权。
>
> 可以紧接着执行`write`操作，不用再等待3秒钟了，当调度器执行`write`会先注册一个写事件，回调函数是让当前协程唤醒继续执行，然后让协程让出执行权。
>
> 执行完`write`可以紧接着执行`read`，在调度器上注册一个读事件，回调函数是当让当前协程唤醒继续执行，让后让协程让出执行权。
>
> 等3秒后，执行定时器的回调函数，让协程继续调用sleep。
>
> 在等待3秒的过程中，一旦可写，那么就继续执行写的协程。
>
> 一旦可读，那么就继续执行读的协程。
>
> 

将`libc`库中的接口重新找回来的方法就是使用`dlsym()`

```c++
#include <dlfcn.h>
/*
 * 第一个参数固定为 RTLD_NEXT，第二个参数为符号的名称
 */
void *dlsym(void *handle, const char *symbol);
```



关于文件管理

`FdCtx`存储每一个`fd`相关的信息，并由`FdManager`管理每一个`FdCtx`，`FdManager`为单例类。这里fd应该是句柄吧。

这部分其实是在`fd_manager.h`文件中，即文件句柄管理类。这里就只是管理，管理文件句柄类型，(是否socket)，是否阻塞,是否关闭,读/写超时时间
```c++
/// 是否初始化
bool m_isInit: 1;
/// 是否socket
bool m_isSocket: 1;
/// 是否hook非阻塞
bool m_sysNonblock: 1;
/// 是否用户主动设置非阻塞
bool m_userNonblock: 1;
/// 是否关闭
bool m_isClosed: 1;
/// 文件句柄
int m_fd;
/// 读超时时间毫秒
uint64_t m_recvTimeout;
/// 写超时时间毫秒
uint64_t m_sendTimeout;
```

构造函数：

```C++
FdCtx::FdCtx(int fd)
    :m_isInit(false)
    ,m_isSocket(false)
    ,m_sysNonblock(false)
    ,m_userNonblock(false)
    ,m_isClosed(false)
    ,m_fd(fd)
    ,m_recvTimeout(-1)
    ,m_sendTimeout(-1) {
    init();
}
bool FdCtx::init() {
    if(m_isInit) {
        return true;
    }
    m_recvTimeout = -1;
    m_sendTimeout = -1;

    struct stat fd_stat;
    if(-1 == fstat(m_fd, &fd_stat)) {
        m_isInit = false;
        m_isSocket = false;
    } else {
        m_isInit = true;
        m_isSocket = S_ISSOCK(fd_stat.st_mode);
    }

    if(m_isSocket) {
        int flags = fcntl_f(m_fd, F_GETFL, 0);
        if(!(flags & O_NONBLOCK)) {
            fcntl_f(m_fd, F_SETFL, flags | O_NONBLOCK);
        }
        m_sysNonblock = true;
    } else {
        m_sysNonblock = false;
    }

    m_userNonblock = false;
    m_isClosed = false;
    return m_isInit;
}
```

setTimeout(设置超时时间)

```c++
void FdCtx::setTimeout(int type, uint64_t v) {
    if(type == SO_RCVTIMEO) {
        m_recvTimeout = v;
    } else {
        m_sendTimeout = v;
    }
}
```

getTimeout(获得超时事件)

```c++
uint64_t FdCtx::getTimeout(int type) {
    if(type == SO_RCVTIMEO) {
        return m_recvTimeout;
    } else {
        return m_sendTimeout;
    }
}
```

FdManager(文件句柄管理类)

```c++
/// 读写锁
RWMutexType m_mutex;
/// 文件句柄集合
std::vector<FdCtx::ptr> m_datas;
```

相当于是一个集合。

构造函数：

```C++
FdManager::FdManager() {
    m_datas.resize(64);
}
```

get(获取/创建文件句柄类FdCtx)：

```c++
FdCtx::ptr FdManager::get(int fd, bool auto_create) {//auto_create：是否自动创建FdCtx
    if(fd == -1) {
        return nullptr;
    }// 集合中没有，并且不自动创建，返回nullptr
    RWMutexType::ReadLock lock(m_mutex);
    if((int)m_datas.size() <= fd) {
        if(auto_create == false) {
            return nullptr;
        }
    } else {
        if(m_datas[fd] || !auto_create) {// 找到了直接返回
            return m_datas[fd];
        }
    }
    lock.unlock();

    RWMutexType::WriteLock lock2(m_mutex);
    FdCtx::ptr ctx(new FdCtx(fd));// 创建新的FdCtx
    if(fd >= (int)m_datas.size()) {// fd比集合下标大，扩充
        m_datas.resize(fd * 1.5);
    }
    m_datas[fd] = ctx;// 放入集合中
    return ctx;
}
```

有则获取，没有则创建。

del(删除文件句柄类）

```c++
void FdManager::del(int fd) {
    RWMutexType::WriteLock lock(m_mutex);
    if((int)m_datas.size() <= fd) {
        return;
    }
    m_datas[fd].reset();
}
```

HOOK模块

将函数接口都存放到`extern "C"`作用域下，指定函数按照C语言的方式进行编译和链接。它的作用是为了解决C++中函数名重载的问题，使得C++代码可以和C语言代码进行互操作。

```c++
//sleep
typedef unsigned int (*sleep_fun)(unsigned int seconds);
extern sleep_fun sleep_f;

typedef int (*usleep_fun)(useconds_t usec);
extern usleep_fun usleep_f;

typedef int (*nanosleep_fun)(const struct timespec *req, struct timespec *rem);
extern nanosleep_fun nanosleep_f;

//socket
...
//read
...
//write
...
```

这里其实是使用 typedef 定义函数指针的用法，前面的`unsigned int`是函数的返回值。然后下面一行使用这个新类型定义了变量sleep_f，后面就可以直接使用这个变量了。

**获取接口原始地址**

使用宏来封装对每个原始接口地址的获取。

将`hook_init()`封装到一个结构体的构造函数中，并创建静态对象，能够在`main`函数运行之前就能将地址保存到函数指针变量当中。

```C++
#define HOOK_FUN(XX) \
    XX(sleep) \
    XX(usleep) \
    XX(nanosleep) \
    XX(socket) \
    XX(connect) \
    XX(accept) \
    XX(read) \
    XX(readv) \
    XX(recv) \
    XX(recvfrom) \
    XX(recvmsg) \
    XX(write) \
    XX(writev) \
    XX(send) \
    XX(sendto) \
    XX(sendmsg) \
    XX(close) \
    XX(fcntl) \
    XX(ioctl) \
    XX(getsockopt) \
    XX(setsockopt)

void hook_init() {
    static bool is_inited = false;
    if(is_inited) {
        return;
    }
#define XX(name) name ## _f = (name ## _fun)dlsym(RTLD_NEXT, #name);
    HOOK_FUN(XX);
#undef XX
}

extern "C" {
#define XX(name) name ## _fun name ## _f = nullptr;
    HOOK_FUN(XX);
#undef XX
}
```

这里的`#undef`是将宏定义取消，防止在后面程序段中用到同样的名字而产生冲突。

解析完分别是

```c++
void hook_init() {
    static bool is_inited = false;
    if (is_inited) {
        return;
    }
    
	sleep_f = (sleep_fun)dlsym(RTLD_NEXT, "sleep");
    usleep_f = (usleep_fun)dlsym(RTLD_NEXT, "usleep");
    ...
    setsocketopt_f = (setsocketopt_fun)dlsym(RTLD_NEXT, "setsocketopt");
}
```

set_hook_enable（设置是否hook）

定义线程局部变量，来控制是否开启hook。

```c++
static thread_local bool t_hook_enable = false;
void set_hook_enable(bool flag) {
    t_hook_enable = flag;
}
```

is_hook_enable（获取是否hook）

```c++
bool is_hook_enable() {
    return t_hook_enable;
}
```

do_io（socket操作真正执行体）

```c++
//socket操作真正执行体 
//fd：文件描述符 fun 原始函数 hook_fun_name	hook的函数名称
//event	事件 timeout_so	超时时间类型 args 可变参数
template<typename OriginFun, typename... Args>
static ssize_t do_io(int fd, OriginFun fun, const char* hook_fun_name,
        uint32_t event, int timeout_so, Args&&... args) {
    if(!sylar::t_hook_enable) {//如果不hook，直接返回原接口
        return fun(fd, std::forward<Args>(args)...);
    }

    sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(fd);// 获取fd对应的FdCtx
    if(!ctx) {// 没有文件 
        return fun(fd, std::forward<Args>(args)...);
    }
    // 看看句柄是否关闭
    if(ctx->isClose()) {
        errno = EBADF;// 坏文件描述符
        return -1;
    }

    if(!ctx->isSocket() || ctx->getUserNonblock()) {// 不是socket 或 用户设置了非阻塞
        return fun(fd, std::forward<Args>(args)...);
    }

    uint64_t to = ctx->getTimeout(timeout_so);// 获得超时时间
    std::shared_ptr<timer_info> tinfo(new timer_info);// 设置超时条件

retry:
    ssize_t n = fun(fd, std::forward<Args>(args)...);// 先执行fun 读数据或写数据 若函数返回值有效就直接返回
    while(n == -1 && errno == EINTR) {// 若中断则重试
        n = fun(fd, std::forward<Args>(args)...);
    }
    if(n == -1 && errno == EAGAIN) {// 若为阻塞状态
        sylar::IOManager* iom = sylar::IOManager::GetThis();// 获得当前IO调度器
        sylar::Timer::ptr timer;// 定时器
        std::weak_ptr<timer_info> winfo(tinfo);// tinfo的弱指针，可以判断tinfo是否已经销毁
        // 说明设置了超时时间
        if(to != (uint64_t)-1) {//添加条件定时器 to时间消息还没来就触发callback
            timer = iom->addConditionTimer(to, [winfo, fd, iom, event]() {
                auto t = winfo.lock();
                if(!t || t->cancelled) {//tinfo失效 || 设了错误  定时器失效了
                    return;
                }
                t->cancelled = ETIMEDOUT;// 没错误的话设置为超时而失败
                iom->cancelEvent(fd, (sylar::IOManager::Event)(event));// 取消事件强制唤醒
            }, winfo);
        }

        int rt = iom->addEvent(fd, (sylar::IOManager::Event)(event));
        if(SYLAR_UNLIKELY(rt)) {// addEvent失败， 取消上面加的定时器
            SYLAR_LOG_ERROR(g_logger) << hook_fun_name << " addEvent("
                << fd << ", " << event << ")";
            if(timer) {
                timer->cancel();
            }
            return -1;
        } else {
            sylar::Fiber::GetThis()->yield();
            if(timer) {
                timer->cancel();
            }
            if(tinfo->cancelled) {// 从定时任务唤醒，超时失败
                errno = tinfo->cancelled;// 从定时任务唤醒，超时失败
                return -1;
            }
            goto retry;// 数据来了就直接重新去操作
        }
    }
    
    return n;
}
```

不太懂mmm，据说可以写同步的方式实现异步的效果。后面涉及到test函数的时候看一下好了。

sleep（睡眠系列）

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

socket（创建socket）

```
int socket(int domain, int type, int protocol) {
    if(!sylar::t_hook_enable) {
        return socket_f(domain, type, protocol);
    }
    int fd = socket_f(domain, type, protocol);
    if(fd == -1) {
        return fd;
    }
    // 将fd放入到文件管理中
    sylar::FdMgr::GetInstance()->get(fd, true);
    return fd;
}
```

connect（socket连接）

```c++
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen) {
    return connect_with_timeout(sockfd, addr, addrlen, sylar::s_connect_timeout);
}
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
    // ----- 异步开始 ----- 先尝试连接
    int n = connect_f(fd, addr, addrlen);
    if(n == 0) {// 连接成功
        return 0;
    } else if(n != -1 || errno != EINPROGRESS) {// 其他错误，EINPROGRESS表示连接操作正在进行中
        return n;
    }

    sylar::IOManager* iom = sylar::IOManager::GetThis();
    sylar::Timer::ptr timer;
    std::shared_ptr<timer_info> tinfo(new timer_info);
    std::weak_ptr<timer_info> winfo(tinfo);

    if(timeout_ms != (uint64_t)-1) {// 设置了超时时间
        timer = iom->addConditionTimer(timeout_ms, [winfo, fd, iom]() {// 加条件定时器
                auto t = winfo.lock();
                if(!t || t->cancelled) {
                    return;
                }
                t->cancelled = ETIMEDOUT;
                iom->cancelEvent(fd, sylar::IOManager::WRITE);
        }, winfo);
    }

    int rt = iom->addEvent(fd, sylar::IOManager::WRITE);// 添加一个写事件
    if(rt == 0) {//只有两种情况唤醒：1. 超时，从定时器唤醒 2. 连接成功，从epoll_wait拿到事件
        sylar::Fiber::GetThis()->yield();
        if(timer) {
            timer->cancel();
        }
        if(tinfo->cancelled) {// 从定时器唤醒，超时失败
            errno = tinfo->cancelled;
            return -1;
        }
    } else {// 添加事件失败
        if(timer) {
            timer->cancel();
        }
        SYLAR_LOG_ERROR(g_logger) << "connect addEvent(" << fd << ", WRITE) error";
    }

    int error = 0;
    socklen_t len = sizeof(int);
    if(-1 == getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len)) {// 获取套接字的错误状态
        return -1;
    }
    if(!error) {// 没有错误，连接成功
        return 0;
    } else {// 有错误，连接失败
        errno = error;
        return -1;
    }
}
```

accept（接收请求）

```c++
int accept(int s, struct sockaddr *addr, socklen_t *addrlen) {
    int fd = do_io(s, accept_f, "accept", sylar::IOManager::READ, SO_RCVTIMEO, addr, addrlen);
    if(fd >= 0) {// 将新创建的连接放到文件管理中
        sylar::FdMgr::GetInstance()->get(fd, true);
    }
    return fd;
}
```

read···（一系列发送与接收）

都是通过`do_io`来实现的

close（关闭socket）

```c++
int close(int fd) {
    if(!sylar::t_hook_enable) {
        return close_f(fd);
    }

    sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(fd);
    if(ctx) {
        auto iom = sylar::IOManager::GetThis();
        if(iom) {// 取消所有事件
            iom->cancelAll(fd);
        }
        sylar::FdMgr::GetInstance()->del(fd);// 在文件管理中删除
    }
    return close_f(fd);
}
```

>有了`hook`模块的加持，在使用IO协程调度器时，如果不想该操作导致整个线程的阻塞，我们可以使用`scheduler`将该任务加入到任务队列中，这样当任务阻塞时，只会使执行该任务的协程挂起，去执行别的任务，在消息来后或者达到超时时间继续执行该协程任务，这样就实现了异步操作。

其实那些回调啊，什么之类的，可能是为了实现异步，或者怎样，感觉自己有点过于关注实现的细节了，具体框架是什么样的，不是很理解了。
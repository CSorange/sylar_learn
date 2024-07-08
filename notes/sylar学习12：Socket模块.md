#  sylar学习12：Socket模块

> 对`socket`的方法进行封装，提供接口方便的创建`TCP`、`UDP`、`Unix`的`socket`对象。当创建一个`socket`对象时，并没有真正的创建一个`socket`句柄，此时它的句柄为`-1`，只有在`bind`、`connect`的时候才会通过`newSock()`创建一个`socket`句柄与对象关联起来，在`accept`时创建新的`socket`对象，并初始化。新的`socket`句柄都初始化为地址复用模式，如果为`TCP`连接，禁用`Nagle`算法，提高传输效率，降低延迟。

类型和协议簇

```c++
enum Type {
        /// TCP类型
        TCP = SOCK_STREAM,
        /// UDP类型
        UDP = SOCK_DGRAM
    };
enum Family {
        /// IPv4 socket
        IPv4 = AF_INET,
        /// IPv6 socket
        IPv6 = AF_INET6,
        /// Unix socket
        UNIX = AF_UNIX,
    };
```

成员变量

```c++
/// socket句柄
int m_sock;
/// 协议簇
int m_family;
/// 类型
int m_type;
/// 协议
int m_protocol;
/// 是否连接
bool m_isConnected;
/// 本地地址
Address::ptr m_localAddress;
/// 远端地址
Address::ptr m_remoteAddress;
```

根据地址创建TCP/UCP套接字

```c++
Socket::ptr Socket::CreateTCP(sylar::Address::ptr address) {
    Socket::ptr sock(new Socket(address->getFamily(), TCP, 0));
    return sock;
}

Socket::ptr Socket::CreateUDP(sylar::Address::ptr address) {
    Socket::ptr sock(new Socket(address->getFamily(), UDP, 0));
    sock->newSock();
    sock->m_isConnected = true;
    return sock;
}

void Socket::newSock() {
    m_sock = socket(m_family, m_type, m_protocol);
    if (SYLAR_LIKELY(m_sock != -1)) {
        initSock();
    } else {
        SYLAR_LOG_ERROR(g_logger) << "socket(" << m_family<< ", " << m_type << ", " << m_protocol << ") errno="<< errno << " errstr=" << strerror(errno);
    }
}
```

UDP这个有点奇怪，感觉又重新socket了一下，不知道为何。

下面是一组排列组合，UDP/TCP，对上了IPv4/IPv6/Unix。

```c++
Socket::ptr Socket::CreateTCPSocket() {
    Socket::ptr sock(new Socket(IPv4, TCP, 0));
    return sock;
}

Socket::ptr Socket::CreateUDPSocket() {
    Socket::ptr sock(new Socket(IPv4, UDP, 0));
    sock->newSock();
    sock->m_isConnected = true;
    return sock;
}

Socket::ptr Socket::CreateTCPSocket6() {
    Socket::ptr sock(new Socket(IPv6, TCP, 0));
    return sock;
}

Socket::ptr Socket::CreateUDPSocket6() {
    Socket::ptr sock(new Socket(IPv6, UDP, 0));
    sock->newSock();
    sock->m_isConnected = true;
    return sock;
}

Socket::ptr Socket::CreateUnixTCPSocket() {
    Socket::ptr sock(new Socket(UNIX, TCP, 0));
    return sock;
}

Socket::ptr Socket::CreateUnixUDPSocket() {
    Socket::ptr sock(new Socket(UNIX, UDP, 0));
    return sock;
}
```

Socket（构造函数）

只是创建对象，没有真正的创建socketfd。

```c++
Socket::Socket(int family, int type, int protocol)
    : m_sock(-1)
    , m_family(family)
    , m_type(type)
    , m_protocol(protocol)
    , m_isConnected(false) {
}
```

~Socket（析构函数）

```c++
Socket::~Socket() {
    close();
}
```

真的很奇怪，不知道这里的close()是指的什么，因为并没有返回值。

getSendTimeout（返回发送超时时间）

```c++
int64_t Socket::getSendTimeout() {
    FdCtx::ptr ctx = FdMgr::GetInstance()->get(m_sock);
    if (ctx) {
        return ctx->getTimeout(SO_SNDTIMEO);
    }
    return -1;
}
```

setSendTimeout（设置发送超时时间）

```c++
void Socket::setSendTimeout(int64_t v) {
    struct timeval tv {
        int(v / 1000), int(v % 1000 * 1000)
    };
    setOption(SOL_SOCKET, SO_SNDTIMEO, tv);
}
```

getRecvTimeout（返回接收超时时间）

```c++
int64_t Socket::getRecvTimeout() {
    FdCtx::ptr ctx = FdMgr::GetInstance()->get(m_sock);
    if (ctx) {
        return ctx->getTimeout(SO_RCVTIMEO);
    }
    return -1;
}
```

setRecvTimeout（设置接收超时时间）

```c++
void Socket::setRecvTimeout(int64_t v) {
    struct timeval tv {
        int(v / 1000), int(v % 1000 * 1000)
    };
    setOption(SOL_SOCKET, SO_RCVTIMEO, tv);
}
```

getOption（获取socketfd信息）

```c++
bool Socket::getOption(int level, int option, void *result, socklen_t *len) {
    int rt = getsockopt(m_sock, level, option, result, (socklen_t *)len);
    if (rt) {
        SYLAR_LOG_DEBUG(g_logger) << "getOption sock=" << m_sock<< " level=" << level << " option=" << option<< " errno=" << errno << " errstr=" << strerror(errno);
        return false;
    }
    return true;
}
int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen) {
    return getsockopt_f(sockfd, level, optname, optval, optlen);
}
```

setOption（设置socketfd信息）

```c++
bool Socket::setOption(int level, int option, const void *result, socklen_t len) {
    if (setsockopt(m_sock, level, option, result, (socklen_t)len)) {
        SYLAR_LOG_DEBUG(g_logger) << "setOption sock=" << m_sock<< " level=" << level << " option=" << option<< " errno=" << errno << " errstr=" << strerror(errno);
        return false;
    }
    return true;
}
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen) {
    if(!sylar::t_hook_enable) {
        return setsockopt_f(sockfd, level, optname, optval, optlen);
    }
    if(level == SOL_SOCKET) {
        if(optname == SO_RCVTIMEO || optname == SO_SNDTIMEO) {
            sylar::FdCtx::ptr ctx = sylar::FdMgr::GetInstance()->get(sockfd);
            if(ctx) {
                const timeval* v = (const timeval*)optval;
                ctx->setTimeout(optname, v->tv_sec * 1000 + v->tv_usec / 1000);
            }
        }
    }
    return setsockopt_f(sockfd, level, optname, optval, optlen);
}
```

这两个函数都有底层函数的身影。

init（初始化socket对象）

```c++
bool Socket::init(int sock) {
    FdCtx::ptr ctx = FdMgr::GetInstance()->get(sock);//获取/创建文件句柄类FdCtx
    if (ctx && ctx->isSocket() && !ctx->isClose()) {
        m_sock        = sock;
        m_isConnected = true;
        initSock();
        getLocalAddress();
        getRemoteAddress();
        return true;
    }
    return false;
}
```

initSock（初始化socketfd）

```c++
void Socket::initSock() {
    int val = 1;
    setOption(SOL_SOCKET, SO_REUSEADDR, val);// SO_REUSEADDR 打开或关闭地址复用功能 option_value不等于0时，打开
    if (m_type == SOCK_STREAM) {
        setOption(IPPROTO_TCP, TCP_NODELAY, val);// Nagle算法通过将小数据块合并成更大的数据块来减少网络传输的次数，提高网络传输的效率。
    }
}
```

newSock（创建socketfd）

```c++
void Socket::newSock() {
    m_sock = socket(m_family, m_type, m_protocol);
    if (SYLAR_LIKELY(m_sock != -1)) {
        initSock();
    } else {
        SYLAR_LOG_ERROR(g_logger) << "socket(" << m_family<< ", " << m_type << ", " << m_protocol << ") errno="<< errno << " errstr=" << strerror(errno);
    }
}
```

listen（监听）

```c++
bool Socket::listen(int backlog) {
    if (!isValid()) {
        SYLAR_LOG_ERROR(g_logger) << "listen error sock=-1";
        return false;
    }
    if (::listen(m_sock, backlog)) {
        SYLAR_LOG_ERROR(g_logger) << "listen error errno=" << errno
                                  << " errstr=" << strerror(errno);
        return false;
    }
    return true;
}
```

accept（接收connect连接）

创建新的`socket`对象，用于与客户端通信。

```c++
Socket::ptr Socket::accept() {
    Socket::ptr sock(new Socket(m_family, m_type, m_protocol));
    int newsock = ::accept(m_sock, nullptr, nullptr);
    if (newsock == -1) {
        SYLAR_LOG_ERROR(g_logger) << "accept(" << m_sock << ") errno="<< errno << " errstr=" << strerror(errno);
        return nullptr;
    }
    if (sock->init(newsock)) {
        return sock;
    }
    return nullptr;
}
```

connect（连接地址）

```c++
bool Socket::connect(const Address::ptr addr, uint64_t timeout_ms) {
    m_remoteAddress = addr;
    if (!isValid()) {
        newSock();
        if (SYLAR_UNLIKELY(!isValid())) {
            return false;
        }
    }

    if (SYLAR_UNLIKELY(addr->getFamily() != m_family)) {
        SYLAR_LOG_ERROR(g_logger) << "connect sock.family("<< m_family << ") addr.family(" << addr->getFamily()<< ") not equal, addr=" << addr->toString();
        return false;
    }

    if (timeout_ms == (uint64_t)-1) {
        if (::connect(m_sock, addr->getAddr(), addr->getAddrLen())) {
            SYLAR_LOG_ERROR(g_logger) << "sock=" << m_sock << " connect(" << addr->toString()<< ") error errno=" << errno << " errstr=" << strerror(errno);
            close();
            return false;
        }
    } else {
        if (::connect_with_timeout(m_sock, addr->getAddr(), addr->getAddrLen(), timeout_ms)) {
            SYLAR_LOG_ERROR(g_logger) << "sock=" << m_sock << " connect(" << addr->toString()<< ") timeout=" << timeout_ms << " error errno="<< errno << " errstr=" << strerror(errno);
            close();
            return false;
        }
    }
    m_isConnected = true;
    getRemoteAddress();
    getLocalAddress();
    return true;
}
```

close（关闭socket）

```c++
bool Socket::close() {
    if (!m_isConnected && m_sock == -1) {
        return true;
    }
    m_isConnected = false;
    if (m_sock != -1) {
        ::close(m_sock);
        m_sock = -1;
    }
    return false;
}
```

send（发送数据：单数据块）

````c++
int Socket::send(const void *buffer, size_t length, int flags) {
    if (isConnected()) {
        return ::send(m_sock, buffer, length, flags);
    }
    return -1;
}
````

send（发送数据：多数据块）

```c++
int Socket::send(const iovec *buffers, size_t length, int flags) {
    if (isConnected()) {
        msghdr msg;
        memset(&msg, 0, sizeof(msg));
        msg.msg_iov    = (iovec *)buffers;
        msg.msg_iovlen = length;
        return ::sendmsg(m_sock, &msg, flags);
    }
    return -1;
}
```

sendTo（指定地址发送数据：单数据块）

```c++
int Socket::sendTo(const void *buffer, size_t length, const Address::ptr to, int flags) {
    if (isConnected()) {
        return ::sendto(m_sock, buffer, length, flags, to->getAddr(), to->getAddrLen());
    }
    return -1;
}
```

sendTo（指定地址发送数据：多数据块）

````c++
int Socket::sendTo(const iovec *buffers, size_t length, const Address::ptr to, int flags) {
    if (isConnected()) {
        msghdr msg;
        memset(&msg, 0, sizeof(msg));
        msg.msg_iov     = (iovec *)buffers;
        msg.msg_iovlen  = length;
        msg.msg_name    = to->getAddr();
        msg.msg_namelen = to->getAddrLen();
        return ::sendmsg(m_sock, &msg, flags);
    }
    return -1;
}
````

recv（接收数据：单数据块）

```c++
int Socket::recv(void *buffer, size_t length, int flags) {
    if (isConnected()) {
        return ::recv(m_sock, buffer, length, flags);
    }
    return -1;
}
```

recv（接收数据：多数据块）

```c++
int Socket::recv(iovec *buffers, size_t length, int flags) {
    if (isConnected()) {
        msghdr msg;
        memset(&msg, 0, sizeof(msg));
        msg.msg_iov    = (iovec *)buffers;
        msg.msg_iovlen = length;
        return ::recvmsg(m_sock, &msg, flags);
    }
    return -1;
}
```

recvFrom（指定地址接收数据：单数据块）

```c++
int Socket::recvFrom(void *buffer, size_t length, Address::ptr from, int flags) {
    if (isConnected()) {
        socklen_t len = from->getAddrLen();
        return ::recvfrom(m_sock, buffer, length, flags, from->getAddr(), &len);
    }
    return -1;
}
```

recvFrom（指定地址接收数据：多数据块）

```c++
int Socket::recvFrom(iovec *buffers, size_t length, Address::ptr from, int flags) {
    if (isConnected()) {
        msghdr msg;
        memset(&msg, 0, sizeof(msg));
        msg.msg_iov     = (iovec *)buffers;
        msg.msg_iovlen  = length;
        msg.msg_name    = from->getAddr();
        msg.msg_namelen = from->getAddrLen();
        return ::recvmsg(m_sock, &msg, flags);
    }
    return -1;
}
```

getRemoteAddress（返回远端地址）

```c++
Address::ptr Socket::getRemoteAddress() {
    if (m_remoteAddress) {
        return m_remoteAddress;
    }

    Address::ptr result;
    switch (m_family) {
    case AF_INET:
        result.reset(new IPv4Address());
        break;
    case AF_INET6:
        result.reset(new IPv6Address());
        break;
    case AF_UNIX:
        result.reset(new UnixAddress());
        break;
    default:
        result.reset(new UnknownAddress(m_family));
        break;
    }
    socklen_t addrlen = result->getAddrLen();
    if (getpeername(m_sock, result->getAddr(), &addrlen)) {// 获取一个已连接套接字的远程地址信息
        SYLAR_LOG_ERROR(g_logger) << "getpeername error sock=" << m_sock<< " errno=" << errno << " errstr=" << strerror(errno);
        return Address::ptr(new UnknownAddress(m_family));
    }
    if (m_family == AF_UNIX) {
        UnixAddress::ptr addr = std::dynamic_pointer_cast<UnixAddress>(result);
        addr->setAddrLen(addrlen);
    }
    m_remoteAddress = result;
    return m_remoteAddress;
}
```

getRemoteAddress（返回本地地址）

```
Address::ptr Socket::getLocalAddress() {
    if (m_localAddress) {
        return m_localAddress;
    }

    Address::ptr result;
    switch (m_family) {
    case AF_INET:
        result.reset(new IPv4Address());
        break;
    case AF_INET6:
        result.reset(new IPv6Address());
        break;
    case AF_UNIX:
        result.reset(new UnixAddress());
        break;
    default:
        result.reset(new UnknownAddress(m_family));
        break;
    }
    socklen_t addrlen = result->getAddrLen();
    if (getsockname(m_sock, result->getAddr(), &addrlen)) {// 获取一个套接字的本地地址信息
        SYLAR_LOG_ERROR(g_logger) << "getsockname error sock=" << m_sock
                                  << " errno=" << errno << " errstr=" << strerror(errno);
        return Address::ptr(new UnknownAddress(m_family));
    }
    if (m_family == AF_UNIX) {
        UnixAddress::ptr addr = std::dynamic_pointer_cast<UnixAddress>(result);
        addr->setAddrLen(addrlen);
    }
    m_localAddress = result;
    return m_localAddress;
}
```

取消读/写/accept/所有事件

```c++
bool Socket::cancelRead() {
    return IOManager::GetThis()->cancelEvent(m_sock, sylar::IOManager::READ);
}

bool Socket::cancelWrite() {
    return IOManager::GetThis()->cancelEvent(m_sock, sylar::IOManager::WRITE);
}

bool Socket::cancelAccept() {
    return IOManager::GetThis()->cancelEvent(m_sock, sylar::IOManager::READ);
}

bool Socket::cancelAll() {
    return IOManager::GetThis()->cancelAll(m_sock);
}
```

test_socket_tcp_client.cc

这里对应的其实是文件test_socket_tcp_client.cc和test_socket_tcp_server.cc

这个其实就是关于socket经典的编程练习。

![image-20240704200120466](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240704200120466.png)

如上图所示。

client：

```c++
void test_tcp_client() {
    int ret;

    auto socket = sylar::Socket::CreateTCPSocket();
    SYLAR_ASSERT(socket);

    auto addr = sylar::Address::LookupAnyIPAddress("0.0.0.0:12345");
    SYLAR_ASSERT(addr);

    ret = socket->connect(addr);
    SYLAR_ASSERT(ret);

    SYLAR_LOG_INFO(g_logger) << "connect success, peer address: " << socket->getRemoteAddress()->toString();

    std::string buffer;
    buffer.resize(1024);
    socket->recv(&buffer[0], buffer.size());
    SYLAR_LOG_INFO(g_logger) << "recv: " << buffer;
    socket->close();

    return;
}
```

可以看到，在客户端同样使用了函数socket创建套接字，使用connect进行连接，不过不同的是在这里客户端时输入接收信息的那一端。在运行`socket->recv(&buffer[0], buffer.size());`时，客户端会等待服务器传入的信息。

对应来看服务器：

```c++
void test_tcp_server() {
    int ret;

    auto addr = sylar::Address::LookupAnyIPAddress("0.0.0.0:12345");
    SYLAR_ASSERT(addr);

    auto socket = sylar::Socket::CreateTCPSocket();
    SYLAR_ASSERT(socket);

    ret = socket->bind(addr); //绑定地址
    SYLAR_ASSERT(ret);
    
    SYLAR_LOG_INFO(g_logger) << "bind success";

    ret = socket->listen();
    SYLAR_ASSERT(ret);

    SYLAR_LOG_INFO(g_logger) << socket->toString() ;
    SYLAR_LOG_INFO(g_logger) << "listening...";

    while(1) {
        auto client = socket->accept();
        SYLAR_ASSERT(client);
        SYLAR_LOG_INFO(g_logger) << "new client: " << client->toString();
        client->send("hello world", strlen("hello world"));
        client->close();
    }
}
```

服务器同样用了socket，bind，listen，accept，并在accept中阻塞，等待客户端的connect。后面它使用send发送消息，并在客户端中显示。

不过这里并没有用多线程的编程方式，而是发送完一次信息就close()，然后再次等待下一次连接。

不过客户端是使用标准的IO协程调度器(即IOManager)来进行的，所以可以看一下整体的流程。

在main函数中：

```c++
sylar::IOManager iom;
iom.schedule(&test_tcp_client);
```

可以看到我们使用了IO协程调度器，想实现test_tcp_client操作。

首先就是构造IOManager。

```c++
IOManager::IOManager(size_t threads, bool use_caller, const std::string &name)
    : Scheduler(threads, use_caller, name)
```

可以看到IOManager是基于Scheduler(协程调度器)的，所以我们先运行Scheduler的构造函数。

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
        ...
    }
```

这里我们需要判断是否把协程调度线程是否纳入调度器，即当前线程。如果纳入的话要`--threads`，这个对后面影响深远。之后创造一个主协程，并设置当前线程的调度器为此Scheduler。

```c++
m_rootFiber.reset(new Fiber(std::bind(&Scheduler::run, this), 0, false));
sylar::Thread::SetName(m_name);
//设置当前线程的主协程为m_rootFiber，这里的m_rootFiber是该线程的主协程（执行run任务的协程），只有默认构造出来的fiber才是主协程
t_scheduler_fiber = m_rootFiber.get();
m_rootThread      = sylar::GetThreadId();// 获得当前线程id
m_threadIds.push_back(m_rootThread);
```

m_rootFiber为调度器所在线程的调度协程，而非主协程。这里我们建一个子协程，并将智能指针指向这个新的协程。这里传入的回调函数run即为协程调度函数。

t_scheduler_fiber当前线程的调度协程即为m_rootFiber。

因为将当前线程纳入调度器，所以把当前线程放入到线程池中去。

```c++
} else {// 不将当前线程纳入调度器
        m_rootThread = -1;
    }
    m_threadCount = threads;
}
```

之后构造IOManager。

```C++
IOManager::IOManager(size_t threads, bool use_caller, const std::string &name)
    : Scheduler(threads, use_caller, name) {
    m_epfd = epoll_create(5000); // 创建一个epollfd
    SYLAR_ASSERT(m_epfd > 0);// 成功时，这些系统调用将返回非负文件描述符。如果出错，则返回-1，并且将errno设置为指示错误。

    int rt = pipe(m_tickleFds);// 创建管道，用于进程间通信
    SYLAR_ASSERT(!rt);// 成功返回0，失败返回-1，并且设置errno。
    ...
```

这里就是创建一个epollfd，然后创建管道。这里的m_epfd，m_tickleFds都是句柄。

```c++
/// epoll 文件句柄
int m_epfd = 0;
/// pipe 文件句柄，fd[0]读端，fd[1]写端
int m_tickleFds[2];
```

```C++
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
```

这里一系列操作让event，m_tickleFds，m_epfd都联系起来了。但没有很看懂这个过程mmm。

然后就是`start()`。

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

但是由于上面提到的，将当前线程纳入调度器，`--threads`，所以这里的m_threadCount为0，就不会进入循环了。

至此`sylar::IOManager iom;`运行完了，下面运行`iom.schedule(&test_tcp_client);`。

这个函数其实很简单，就是添加调度任务。

```c++
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

其中主要就是函数`scheduleNoLock`

```c++
bool scheduleNoLock(FiberOrCb fc, int thread) {
        bool need_tickle = m_tasks.empty();
        ScheduleTask task(fc, thread);
        if (task.fiber || task.cb) {
            m_tasks.push_back(task);
        }
        return need_tickle;
    }
```

ScheduleTask为任务队列中的一个元素，可以是一个协程，也可以一个回调函数，兼容两种存在形式。

`iom.schedule(&test_tcp_client);`结束之后就运行IOManager的析构函数。

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

首先是函数`stop()`

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
	...
```

我们查看当前协程，主协程，调度协程：

![image-20240704211009354](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240704211009354.png)

可以看到现在还是主协程。

在`m_rootFiber->resume()`中会切换协程到m_rootFiber。

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

进来之后运行`SetThis(this);`，t_fiber会变为m_rootFiber，但实际的运行内容还没有变，需要通过swapcontext进行切换。需要注意的是，前面是旧协程，后面是新协程，所以相当于我们需要保存主协程，然后运行。因为进来是`m_rootFiber->resume();`所以我们之后会进到协程m_rootFiber的上下文。因为之前构造fiber时，makecontext函数初始化一个ucontext_t后还指明了该context的入口函数。

`makecontext(&m_ctx, &Fiber::MainFunc, 0); `

可以看到入口函数是MainFunc，所以我们这次翻转之后也会进入这个函数。

```C++
void Fiber::MainFunc() {
    Fiber::ptr cur = GetThis(); // 获得当前进程，GetThis()的shared_from_this()方法让引用计数加1
    SYLAR_ASSERT(cur);
    //执行任务
    cur->m_cb();
    ...
```

之后进入回调函数。往前找会发现：`m_rootFiber.reset(new Fiber(std::bind(&Scheduler::run, this), 0, false));`即回调函数为run，这也是我们实际要进行的部分。

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
```

这里定义的idle_fiber是任务执行完进行的(目前之前来看)，

```c++
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
```

我们遍历m_tasks调度任务队列(其实我们知道，这个队列里只有一个)。拿出来并将这个从队列里删除，然后处理这个任务。

```c++
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
```

这里有两种情况(第三种感觉不怎么会出现，不看了mmm)一种是任务为一个协程，那么就直接调用resume函数。另一种任务不为一个协程，为一个回调函数，那么就包装成一个协程，调用resume函数。

在这里运行`SetThis(this);`之后，会发现此时的协程id和t_fiber和t_thread_fiber都不一样，这是合理的，因为是根据任务新创建的协程。

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

因为前面生成协程为`new Fiber(task.cb)`，同时m_runInScheduler默认值为true，所以是和调度器的主协程进行swap。翻转之后执行的才是新生成协程的内容。

然后就又来了MainFunc，每次翻转之后都会进入这里，然后就又要执行回调函数。此时就是进行的新协程的回调函数，即任务，即test_tcp_client()。

整个运行过程我们不看，直接跳到最后。

然后就是执行完这个任务，回到MainFunc，平稳的运行到最后，进行yield函数。

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

此时如果看t_fiber，可以看到此时仍为新协程，`SetThis(t_thread_fiber.get());`之后自然变为了主协程。

![image-20240704221910735](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240704221910735.png)

因为我们在生成此协程的时候默认是参加调度器调度，所以还是会进行`swapcontext(&m_ctx, &(Scheduler::GetMainFiber()->m_ctx))`。所以就把主协程存下了，然后运行调度器的主协程，即m_rootFiber。

因为上次m_rootFiber进行的是和新协程的互换，运行到当时这个位置，然后退出resume()函数。然后就回到了新协程调用resume函数的位置，当时还在run函数中，然后我们到下一行继续运行。

这里有个问题，他跑到任务是协程的resume函数的下一行了，还挺奇怪的，不过倒也不影响整体的格局。

然后因为tasks中只有这一个任务，所以我们就进入到了最后一重else。

```c++
else {
            // 进到这个分支情况一定是任务队列空了，调度idle协程即可
            if (idle_fiber->getState() == Fiber::TERM) {
                // 如果调度器没有调度任务，那么idle协程会不停地resume/yield，不会结束，如果idle协程结束了，那一定是调度器停止了
                SYLAR_LOG_DEBUG(g_logger) << "idle fiber term";
                break;
            }
            ++m_idleThreadCount;
            idle_fiber->resume();
            --m_idleThreadCount;
```

所以上面说的不太对，这里是会用到，或者一定会用到的。

然后进入idle_fiber的resume函数，这个函数在运行完task中所有任务一定会运行的。

然后就是又进入到MainFunc函数，这次进入idle_fiber的回调函数。



总结一下吧，其实从上层的角度来看可以很简单，就是一个协程切换的过程。一开始是主协程，然后要干活了，切换到调度主协程，用调度主协程->resume()来实现。然后有一个时间差，在翻转之前，这段时间修改一下t_fiber。然后翻转之火就开始运行调度主协程了。

MainFunc相当于切换协程后的入口，m_cb是我们主要想做的内容。做完之后在MainFunc中会进行yield，即回到主协程，注意这里的翻转和resume中的是相反的。在进行yield之前，如果是调度器调度的任务，则回到调度主协程，如果是子协程，则回到主协程，一切都很有序。

然后中间有个地方自己不是很理解，就是run中的第三种情况，运行idle时。这里进入了IOManager的idle函数中。但这个也是和epoll，和管道相关的地方，自己之后还要好好看看。另外就是为什么进入的时IOManager的idle，自己也不是很清楚。




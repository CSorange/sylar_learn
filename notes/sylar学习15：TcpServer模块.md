# sylar学习15：TcpServer模块

概述：

> 封装了一个`TCP`服务器，然后基于`TcpServer`实现了一个`EchoServer`。

## *class* TcpServer

成员变量：

```c++
/// 监听Socket数组
std::vector<Socket::ptr> m_socks;
/// 新连接的Socket工作的调度器
IOManager* m_ioWorker;
/// 服务器Socket接收连接的调度器
IOManager* m_acceptWorker;
/// 接收超时时间(毫秒)
uint64_t m_recvTimeout;
/// 服务器名称
std::string m_name;
/// 服务器类型
std::string m_type;
/// 服务是否停止
bool m_isStop;
```

TcpServer（构造函数）

```c++
TcpServer::TcpServer(sylar::IOManager* io_worker,
                    sylar::IOManager* accept_worker)
    :m_ioWorker(io_worker)
    ,m_acceptWorker(accept_worker)
    ,m_recvTimeout(g_tcp_server_read_timeout->getValue())
    ,m_name("sylar/1.0.0")
    ,m_type("tcp")
    ,m_isStop(true) {
}
```

其实就是成员变量的赋值。

~TcpServer（析构函数）

```c++
TcpServer::~TcpServer() {
    for(auto& i : m_socks) {
        i->close();
    }
    m_socks.clear();
}
```

主要处理的监听Socket数组，关闭数组中的socket，清除Socket数组。

bind（绑定地址以及监听），可以获取绑定失败的地址。

```C
bool TcpServer::bind(sylar::Address::ptr addr) {
    std::vector<Address::ptr> addrs;
    std::vector<Address::ptr> fails;
    addrs.push_back(addr);
    return bind(addrs, fails);
}

bool TcpServer::bind(const std::vector<Address::ptr>& addrs
                        ,std::vector<Address::ptr>& fails ) {
    for(auto& addr : addrs) {
        Socket::ptr sock = Socket::CreateTCP(addr);// 创建TCPsocket
        if(!sock->bind(addr)) {// bind
            SYLAR_LOG_ERROR(g_logger) << "bind fail errno="
                << errno << " errstr=" << strerror(errno)
                << " addr=[" << addr->toString() << "]";
            fails.push_back(addr); // bind失败放入失败数组
            continue;
        }
        if(!sock->listen()) {// 监听
            SYLAR_LOG_ERROR(g_logger) << "listen fail errno="
                << errno << " errstr=" << strerror(errno)
                << " addr=[" << addr->toString() << "]";
            fails.push_back(addr);// 监听失败放入失败数组
            continue;
        }
        m_socks.push_back(sock);
    }

    if(!fails.empty()) {// 有绑定失败的地址，清空监听socket数组
        m_socks.clear();
        return false;
    }

    for(auto& i : m_socks) {// 绑定成功
        SYLAR_LOG_INFO(g_logger) << "type=" << m_type
            << " name=" << m_name
            << " server bind success: " << *i;
    }
    return true;
}
```

这里提供两种接口，一个是绑定一个地址，看看能不能成功，另一个是绑定一些列地址，看看能不能全部成功，如果不难，则也储存那些失败的地址。

过程其实很好理解，就是逐个地址进行绑定，然后监听，之后进行判断，然后储存绑定失败的地址。

startAccept（开始接受连接）

```c++
void TcpServer::startAccept(Socket::ptr sock) {
    while(!m_isStop) {// 服务器不停止，一直接收连接
        Socket::ptr client = sock->accept();// accept
        if(client) {// 连接成功
            client->setRecvTimeout(m_recvTimeout);// 设置接收超时时间
            m_ioWorker->schedule(std::bind(&TcpServer::handleClient,
                        shared_from_this(), client));// handleClient结束之前， TcpServer不能结束，shared_from_this，把自己传进去
        } else {
            SYLAR_LOG_ERROR(g_logger) << "accept errno=" << errno
                << " errstr=" << strerror(errno);
        }
    }
}
```

start（启动服务）

```c++
bool TcpServer::start() {
    if(!m_isStop) {
        return true;
    }
    m_isStop = false;
    for(auto& sock : m_socks) {// 每个socket接收连接任务放入任务队列中
        m_acceptWorker->schedule(std::bind(&TcpServer::startAccept,
                    shared_from_this(), sock));
    }
    return true;
}
```

stop（停止服务器）

```c++
void TcpServer::stop() {
    m_isStop = true;
    auto self = shared_from_this();
    m_acceptWorker->schedule([this, self]() {// 将this和self作为参数传递给异步任务的lambda函数，以确保异步任务执行期间当前对象的shared_ptr一直有效
        for(auto& sock : m_socks) {
            sock->cancelAll();
            sock->close();
        }
        m_socks.clear();
    });
}
```



test_tcp_server
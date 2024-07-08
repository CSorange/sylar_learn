# sylar学习16：Socket流接口模块

> 基于序列化模块封装了读和写操作，但是`socket`的`API`并不保证一定能够写或读到规定的字节数，所以封装了`readFixSize`、`writeFixSize`保证一定操作规定字节的数据。

该模块主要有以下几类：

- **class Stream**：基类，抽象类。
- **class SocketStream**：内部封装一个`socket`。

## *class* Stream

这里定义了五个纯虚函数和四个虚函数。

```c++
virtual int read(void* buffer, size_t length) = 0;
virtual int read(ByteArray::ptr ba, size_t length) = 0;
virtual int write(const void* buffer, size_t length) = 0;
virtual int write(ByteArray::ptr ba, size_t length) = 0;
virtual void close() = 0;
```

其中有一个关闭流，剩下四个两个读，两个写，从buffer中或者从ByteArray

还有四个虚函数，这四个都是读取/写固定长度的。

```c++
int Stream::readFixSize(void* buffer, size_t length) {
    size_t offset = 0;// 偏移量
    int64_t left = length;// 剩余字节数
    while(left > 0) {// 若没读完
        int64_t len = read((char*)buffer + offset, left);// 读数据
        if(len <= 0) {// 读取失败
            return len;
        }
        offset += len;// 更新偏移量
        left -= len;// 更新剩余字节数
    }
    return length;
}

int Stream::readFixSize(ByteArray::ptr ba, size_t length) {
    int64_t left = length;
    while(left > 0) {
        int64_t len = read(ba, left);
        if(len <= 0) {
            return len;
        }
        left -= len;
    }
    return length;
}

int Stream::writeFixSize(const void* buffer, size_t length) {
    size_t offset = 0;
    int64_t left = length;
    while(left > 0) {
        int64_t len = write((const char*)buffer + offset, left);
        if(len <= 0) {
            return len;
        }
        offset += len;
        left -= len;
    }
    return length;

}

int Stream::writeFixSize(ByteArray::ptr ba, size_t length) {
    int64_t left = length;
    while(left > 0) {
        int64_t len = write(ba, left);
        if(len <= 0) {
            return len;
        }
        left -= len;
    }
    return length;
}
```

其实这里也调用了纯虚函数。有点奇怪的是，如果不是固定长度的也是需要传入长度参数值。可能read/write函数是要读多少长度，不一定可以读到这么长，所以需要read/writeFixSize进行封装。

嗷嗷这是照应最开始的那段简介。

## *class* SocketStream

成员变量：

```C++
/// Socket类
Socket::ptr m_socket;
/// 是否主控
bool m_owner;
```

构造函数：

```c++
SocketStream(Socket::ptr sock, bool owner = true);
SocketStream::SocketStream(Socket::ptr sock, bool owner)
    :m_socket(sock)
    ,m_owner(owner) {
}
```

其实就是对两个成员函数的初始化。

析构函数：关闭socket。

```c++
SocketStream::~SocketStream() {
    if(m_owner && m_socket) {
        m_socket->close();
    }
}
```

isConnected（连接状态）

```c++
bool SocketStream::isConnected() const {
    return m_socket && m_socket->isConnected();
}
```

close（关闭socket）

```c++
void SocketStream::close() {
    if(m_socket) {
        m_socket->close();
    }
}
```

hh这个和析构函数几乎一模一样。

isConnected（连接状态）

```c++
bool SocketStream::isConnected() const {
    return m_socket && m_socket->isConnected();
}
```

read（读数据，读取到内存）

```c++
int SocketStream::read(void* buffer, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    return m_socket->recv(buffer, length);
}
```

读到内存其实没有很难，其实这个就可以理解为读到指定位置。

read（读数据，读取到ByteArray）

```c++
int SocketStream::read(ByteArray::ptr ba, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    std::vector<iovec> iovs;
    ba->getWriteBuffers(iovs, length); // 获取可写入的缓存,保存成iovec数组
    int rt = m_socket->recv(&iovs[0], iovs.size()); //准备接受
    if(rt > 0) {
        ba->setPosition(ba->getPosition() + rt);
    }
    return rt;
}
```

关于这里的read和write，read是读，相当于接收数据。write是写，相当于是发送数据。

这里借用ByteArray，我们首先活动可写入的缓存，保存成iovec数组，然后再根据iovec数组接收数据。

这里关于recv和send的底层指令可以看之前那个服务器和客户端的例子。recv之后会停等，然后等待send数据。

接着是关于write的函数：

```c++
int SocketStream::write(const void* buffer, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    return m_socket->send(buffer, length);
}

int SocketStream::write(ByteArray::ptr ba, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    std::vector<iovec> iovs;
    ba->getReadBuffers(iovs, length);
    int rt = m_socket->send(&iovs[0], iovs.size());
    if(rt > 0) {
        ba->setPosition(ba->getPosition() + rt);
    }
    return rt;
}
```

模块还封装了一些其他关于socket的函数：

```c++
Address::ptr getRemoteAddress();
Address::ptr getLocalAddress();
std::string getRemoteAddressString();
std::string getLocalAddressString();
```

这里就不做展开了。
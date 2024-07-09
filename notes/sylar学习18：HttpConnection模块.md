# sylar学习18：HttpConnection模块

HttpConnection模块概述：

>`class HttpConnection`继承自`class SocketStream`，发送请求报文，接收响应报文。
>
>封装`struct HttpResult`HTTP响应结果
>
>实现连接池`class HttpConnectionPool`，仅在长连接有效时使用。
>
>封装`class URI`，使用状态机解析`URI`。

URI格式：

```
foo://user@example.com:8042/over/there?name=ferret#nose
_/   ___________________/_________/ _________/ __/
 |              |              |            |        |
scheme      authority         path        query   fragment
 |   __________________________|__
/ \ /                             \
urn:example:animal:ferret:nose

authority   = [ userinfo "@" ] host [ ":" port ]
```

### *class* Uri

在有限状态机action中设置保存解析出来的字段。

成员变量：

```c++
/// schema
std::string m_scheme;
/// 用户信息
std::string m_userinfo;
/// host
std::string m_host;
/// 路径
std::string m_path;
/// 查询参数
std::string m_query;
/// fragment
std::string m_fragment;
/// 端口
int32_t m_port;
```

Uri构造函数

```c++
Uri::Uri()
    : m_port(0) {}
```

Create（创建URI）

```c++
Uri::ptr Uri::Create(const std::string &urlstr) {
    Uri::ptr uri(new Uri);
    struct http_parser_url parser;

    http_parser_url_init(&parser);

    if (http_parser_parse_url(urlstr.c_str(), urlstr.length(), 0, &parser) != 0) {
        return nullptr;
    }

    if (parser.field_set & (1 << UF_SCHEMA)) {
        uri->setScheme(
            std::string(urlstr.c_str() + parser.field_data[UF_SCHEMA].off,
                        parser.field_data[UF_SCHEMA].len));
    }

    if (parser.field_set & (1 << UF_USERINFO)) {
        uri->setUserinfo(
            std::string(urlstr.c_str() + parser.field_data[UF_USERINFO].off,
                        parser.field_data[UF_USERINFO].len));
    }

    if (parser.field_set & (1 << UF_HOST)) {
        uri->setHost(
            std::string(urlstr.c_str() + parser.field_data[UF_HOST].off,
                        parser.field_data[UF_HOST].len));
    }

    if (parser.field_set & (1 << UF_PORT)) {
        uri->setPort(std::stoi(
            std::string(urlstr.c_str() + parser.field_data[UF_PORT].off,
                        parser.field_data[UF_PORT].len)));
    } else {
        //默认端口号解析只支持http/ws/https
        if (uri->getScheme() == "http" || uri->getScheme() == "ws") {
            uri->setPort(80);
        } else if (uri->getScheme() == "https") {
            uri->setPort(443);
        }
    }

    if (parser.field_set & (1 << UF_PATH)) {
        uri->setPath(
            std::string(urlstr.c_str() + parser.field_data[UF_PATH].off,
                        parser.field_data[UF_PATH].len));
    }

    if (parser.field_set & (1 << UF_QUERY)) {
        uri->setQuery(
            std::string(urlstr.c_str() + parser.field_data[UF_QUERY].off,
                        parser.field_data[UF_QUERY].len));
    }

    if (parser.field_set & (1 << UF_FRAGMENT)) {
        uri->setFragment(
            std::string(urlstr.c_str() + parser.field_data[UF_FRAGMENT].off,
                        parser.field_data[UF_FRAGMENT].len));
    }

    return uri;
}
```

相当于在给uri中的其他变量赋值。

isDefaultPort（是否为默认端口：80，443）

```c++
bool Uri::isDefaultPort() const {
    if (m_scheme == "http" || m_scheme == "ws") {
        return m_port == 80;
    } else if (m_scheme == "https") {
        return m_port == 443;
    } else {
        return false;
    }
}
```

getPort（返回端口号）

```c++
int32_t Uri::getPort() const {
    if (m_port) {
        return m_port;
    }
    if (m_scheme == "http" || m_scheme == "ws") {
        return 80;
    } else if (m_scheme == "https" || m_scheme == "wss") {
        return 443;
    }
    return m_port;
}
```

getPath（返回路径）

```c++
const std::string &Uri::getPath() const {
    static std::string s_default_path = "/";
    return m_path.empty() ? s_default_path : m_path;
}
```

createAddress（创建地址）

```c++
Address::ptr Uri::createAddress() const {
    auto addr = Address::LookupAnyIPAddress(m_host);
    if(addr) {
        addr->setPort(getPort());
    }
    return addr;
}
```

嗷所以其实是得到一个IP地址，然后再加上端口，这样就全乎了。



### *class* HttpResult

HTTP响应结果

```c++
enum class Error {
        /// 正常
        OK = 0,
        /// 非法URL
        INVALID_URL = 1,
        /// 无法解析HOST
        INVALID_HOST = 2,
        /// 连接失败
        CONNECT_FAIL = 3,
        /// 连接被对端关闭
        SEND_CLOSE_BY_PEER = 4,
        /// 发送请求产生Socket错误
        SEND_SOCKET_ERROR = 5,
        /// 超时
        TIMEOUT = 6,
        /// 创建Socket失败
        CREATE_SOCKET_ERROR = 7,
        /// 从连接池中取连接失败
        POOL_GET_CONNECTION = 8,
        /// 无效的连接
        POOL_INVALID_CONNECTION = 9,
    };
```

成员变量：

```c++
/// 错误码
int result;
/// HTTP响应结构体
HttpResponse::ptr response;
/// 错误描述
std::string error;
/// 转字符串
std::string toString() const;
```

构造函数：

```c++
HttpResult(int _result
               ,HttpResponse::ptr _response
               ,const std::string& _error)
        :result(_result)
        ,response(_response)
        ,error(_error) {}
```

即对成员变量进行赋值。

## *class* HttpConnection

成员变量：

```c++
/// 创建时间
uint64_t m_createTime = 0;
/// 该连接已使用的次数，只在使用连接池的情况下有用
uint64_t m_request = 0;
```

构造函数：

```c++
HttpConnection::HttpConnection(Socket::ptr sock, bool owner)
    :SocketStream(sock, owner) {
}
```

这里其实就是进行流赋值。

recvResponse（接收响应报文）

```c++
HttpResponse::ptr HttpConnection::recvResponse() {
    HttpResponseParser::ptr parser(new HttpResponseParser);
    uint64_t buff_size = HttpRequestParser::GetHttpRequestBufferSize();
    std::shared_ptr<char> buffer(// 智能指针接管
            new char[buff_size + 1], [](char* ptr){
                delete[] ptr;
            });
    char* data = buffer.get();
    int offset = 0;
    do {// 在offset后面接着读数据
        int len = read(data + offset, buff_size - offset);
        if(len <= 0) {
            close();
            return nullptr;
        }
        len += offset;// 当前已经读取的数据长度
        data[len] = '\0';
        size_t nparse = parser->execute(data, len);// 解析缓冲区data中的数据,execute会将data向前移动nparse个字节，nparse为已经成功解析的字节数
        if(parser->hasError()) {
            close();
            return nullptr;
        }
        offset = len - nparse;// 此时data还剩下len - nparse个字节
        if(offset == (int)buff_size) {// 缓冲区满了还没解析完
            close();
            return nullptr;
        }
        if(parser->isFinished()) {// 解析结束
            break;
        }
    } while(true);//这里改为不支持chunked了，不过估计支持了自己也看不太懂mmm
    return parser->getData();
}
```

sendRequest（发送请求报文）

```c++
int HttpConnection::sendRequest(HttpRequest::ptr rsp) {
    std::stringstream ss;
    ss << *rsp;
    std::string data = ss.str();
    return writeFixSize(data.c_str(), data.size());
}
```

DoRequest （发送HTTP请求）

最后所有的请求都是由这个方法实现的。

```c++
HttpResult::ptr HttpConnection::DoRequest(HttpMethod method
                            , Uri::ptr uri
                            , uint64_t timeout_ms
                            , const std::map<std::string, std::string>& headers
                            , const std::string& body) {
    HttpRequest::ptr req = std::make_shared<HttpRequest>();// 创建http请求报文
    req->setPath(uri->getPath());// 设置path
    req->setQuery(uri->getQuery());// 设置qurey
    req->setFragment(uri->getFragment());// 设置fragment
    req->setMethod(method);// 设置请求方法
    bool has_host = false;// 是否有主机号
    for(auto& i : headers) {
        if(strcasecmp(i.first.c_str(), "connection") == 0) {
            if(strcasecmp(i.second.c_str(), "keep-alive") == 0) {
                req->setClose(false);
            }
            continue;
        }
        // 看有没有设置host
        if(!has_host && strcasecmp(i.first.c_str(), "host") == 0) {
            has_host = !i.second.empty();
        }
        // 设置header
        req->setHeader(i.first, i.second);
    }
    if(!has_host) {//若没有host，则用uri的host
        req->setHeader("Host", uri->getHost());
    }
    req->setBody(body);// 设置body
    return DoRequest(req, uri, timeout_ms);
}
HttpResult::ptr HttpConnection::DoRequest(HttpMethod method
                            , const std::string& url
                            , uint64_t timeout_ms
                            , const std::map<std::string, std::string>& headers
                            , const std::string& body) {
    Uri::ptr uri = Uri::Create(url);
    if(!uri) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::INVALID_URL
                , nullptr, "invalid url: " + url);
    }
    return DoRequest(method, uri, timeout_ms, headers, body);
}
```

这两个本质上是一致的。

```c++
HttpResult::ptr HttpConnection::DoRequest(HttpRequest::ptr req
                            , Uri::ptr uri
                            , uint64_t timeout_ms) {
    Address::ptr addr = uri->createAddress();// 通过uri创建address
    if(!addr) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::INVALID_HOST
                , nullptr, "invalid host: " + uri->getHost());
    }
    Socket::ptr sock = Socket::CreateTCP(addr);// 创建TCPsocket
    if(!sock) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::CREATE_SOCKET_ERROR
                , nullptr, "create socket fail: " + addr->toString()
                        + " errno=" + std::to_string(errno)
                        + " errstr=" + std::string(strerror(errno)));
    }
    if(!sock->connect(addr)) {// 发起请求连接
        return std::make_shared<HttpResult>((int)HttpResult::Error::CONNECT_FAIL
                , nullptr, "connect fail: " + addr->toString());
    }
    sock->setRecvTimeout(timeout_ms);// 设置接收超时时间
    HttpConnection::ptr conn = std::make_shared<HttpConnection>(sock);// 创建httpconnection
    int rt = conn->sendRequest(req);// 发送请求报文
    if(rt == 0) {// 若为0，则表示远端关闭连接
        return std::make_shared<HttpResult>((int)HttpResult::Error::SEND_CLOSE_BY_PEER
                , nullptr, "send request closed by peer: " + addr->toString());
    }
    if(rt < 0) {// 小于0，失败
        return std::make_shared<HttpResult>((int)HttpResult::Error::SEND_SOCKET_ERROR
                    , nullptr, "send request socket error errno=" + std::to_string(errno)
                    + " errstr=" + std::string(strerror(errno)));
    }
    auto rsp = conn->recvResponse();// 接收响应报文
    if(!rsp) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::TIMEOUT
                    , nullptr, "recv response timeout: " + addr->toString()
                    + " timeout_ms:" + std::to_string(timeout_ms));
    }
    return std::make_shared<HttpResult>((int)HttpResult::Error::OK, rsp, "ok");// 结果成功，返回响应报文
}
```



### *class* HttpConnectionPool

> 连接池，仅在长连接时有效，`Connection: keep-alive`

```c++
/// Host字段默认值
std::string m_host;
/// Host字段默认值，m_vhost优先级高于m_host
std::string m_vhost;
/// 端口
uint32_t m_port;
/// 暂未使用
uint32_t m_maxSize;
/// 单个连接的最大存活时间
uint32_t m_maxAliveTime;
/// 单个连接的最大复用次数
uint32_t m_maxRequest;
/// 互斥锁
MutexType m_mutex;
/// 连接池，链表形式存储
std::list<HttpConnection*> m_conns;
/// 当前连接池的可用连接数量
std::atomic<int32_t> m_total = {0};
```

构造函数：

```c++
HttpConnectionPool::HttpConnectionPool(const std::string& host
,const std::string& vhost,uint32_t port,uint32_t max_size,uint32_t max_alive_time,uint32_t max_request)
    :m_host(host)
    ,m_vhost(vhost)
    ,m_port(port)
    ,m_maxSize(max_size)
    ,m_maxAliveTime(max_alive_time)
    ,m_maxRequest(max_request) {
}
```

是的，还是对成员变量进行赋值。

getConnection（获得连接）

```c++
HttpConnection::ptr HttpConnectionPool::getConnection() {
    uint64_t now_ms = sylar::GetCurrentMS();
    std::vector<HttpConnection*> invalid_conns;// 非法的连接
    HttpConnection* ptr = nullptr;
    MutexType::Lock lock(m_mutex);
    while(!m_conns.empty()) {// 若连接池不为空
        auto conn = *m_conns.begin();// 取出第一个connection
        m_conns.pop_front();// 弹掉
        if(!conn->isConnected()) { // 不在连接状态，放入非法vec中
            invalid_conns.push_back(conn);
            continue;
        }
        if((conn->m_createTime + m_maxAliveTime) > now_ms) {// 已经超过了最大连接时间，放入非法vec中
            invalid_conns.push_back(conn);
            continue;
        }
        ptr = conn;// 获得当前connection
        break;
    }
    lock.unlock();
    for(auto i : invalid_conns) {// 将非法con删除
        delete i;
    }
    m_total -= invalid_conns.size();// 更新连接总数

    if(!ptr) {// 若没有连接
        IPAddress::ptr addr = Address::LookupAnyIPAddress(m_host);// 根据host创建addr
        if(!addr) {
            SYLAR_LOG_ERROR(g_logger) << "get addr fail: " << m_host;
            return nullptr;
        }
        addr->setPort(m_port);// 设置端口号
        Socket::ptr sock = Socket::CreateTCP(addr);// 创建TCPsocket
        if(!sock) {
            SYLAR_LOG_ERROR(g_logger) << "create sock fail: " << *addr;
            return nullptr;
        }
        if(!sock->connect(addr)) {// 连接
            SYLAR_LOG_ERROR(g_logger) << "sock connect fail: " << *addr;
            return nullptr;
        }

        ptr = new HttpConnection(sock);
        ++m_total;
    }
    return HttpConnection::ptr(ptr,std::bind(&HttpConnectionPool::ReleasePtr, std::placeholders::_1, this));

}
```

ReleasePtr（释放connection）

```c++
void HttpConnectionPool::ReleasePtr(HttpConnection* ptr, HttpConnectionPool* pool) {
    ++ptr->m_request;
    if(!ptr->isConnected()// 已经关闭了链接，超时，超过最大请求数量
            || ((ptr->m_createTime + pool->m_maxAliveTime) >= sylar::GetCurrentMS())
            || (ptr->m_request >= pool->m_maxRequest)) {
        delete ptr;
        --pool->m_total;
        return;
    }
    MutexType::Lock lock(pool->m_mutex);//在这行之前不进行++ptr->m_request;么，不太一样mmm
    pool->m_conns.push_back(ptr);// 重新放入连接池中
}
```


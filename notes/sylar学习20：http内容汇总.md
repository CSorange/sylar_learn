# sylar学习20：http内容汇总

这里汇总一下关于http的内容吧，其实后面几个章节都是在讲这个东西。这里我们通过test文件来进行串联。

### test_http.cc

首先是http.c/h中的内容。

这里其实主要介绍了两个类：HttpRequest(http请求结构)和HttpResponse(http应答结构)。

其此还有两个类：HttpMethod(http方法枚举)和HttpStatus(http状态枚举)，这两个类主要是保存一些状态码和数字的对应关系，与请求/应答结构相对应。

这里面主要的函数就是dump函数，函数的作用为序列化输出到流。在HttpRequest和HttpResponse均有这个函数。我们在对应的检测函数test_http.cc中也可以看到。

```c++
void test_http_response() {
    sylar::http::HttpResponse rsp;
    rsp.setStatus(sylar::http::HttpStatus::OK);
    rsp.setHeader("Content-Type", "text/html");
    rsp.setBody("<!DOCTYPE html>"
                "<html>"
                "<head>"
                "<title>hello world</title>"
                "</head>"
                "<body><p>hello world</p></body>"
                "</html>");
    rsp.setCookie("kookie1", "value1", 0, "/");
    rsp.setCookie("kookie2", "value2", 0, "/");
    rsp.dump(std::cout);
}
```

在应答函数检测中，使用若干set函数之后用dump进行序列化输出。

```c++
void test_http_request() {
    sylar::http::HttpRequest req;
    req.setMethod(sylar::http::HttpMethod::GET);
    req.setVersion(0x11);
    req.setPath("/search");
    req.setQuery("q=url+%E5%8F%82%E6%95%B0%E6%9E%84%E9%80%A0&oq=url+%E5%8F%82%E6%95%B0%E6%9E%84%E9%80%A0+&aqs=chrome..69i57.8307j0j7&sourceid=chrome&ie=UTF-8");
    req.setHeader("Accept", "text/plain");
    req.setHeader("Content-Type", "application/x-www-form-urlencoded");
    req.setHeader("Cookie", "yummy_cookie=choco; tasty_cookie=strawberry");
    req.setBody("title=test&sub%5B1%5D=1&sub%5B2%5D=2"); // title=test&sub[1]=1&sub[2]=2

    req.dump(std::cout);

    std::cout << std::endl;
    std::cout << req.getParam("q") << std::endl;
    std::cout << req.getParam("title") << std::endl;
    std::cout << req.getParam("sub[1]") << std::endl;
    std::cout << req.getParam("sub[2]") << std::endl;
    std::cout << req.getHeader("Accept") << std::endl;
    std::cout << req.getCookie("yummy_cookie") << std::endl;
    std::cout << req.getCookie("tasty_cookie") << std::endl;
    std::cout << std::endl;
}
```

在请求函数检测中采用同样的方式，不过比应答函数检测多了对get系列函数的检测。



### test_http_parser.cc

首先这个函数还是进行了命令行/配置文件的处理的。

```c++
int main(int argc, char *argv[]) {
    sylar::EnvMgr::GetInstance()->init(argc, argv);
   sylar::Config::LoadFromConfDir(sylar::EnvMgr::GetInstance()->getConfigPath());
    test_request(test_request_data);
    test_request(test_request_chunked_data);
    test_response(test_response_data);
    return 0;
}
```

之后就是函数`test_request`和`test_response`。其中参数test_request_data、test_request_chunked_data、test_response_data都是字符串数组。

这个测试文件对应的文件是http_parse.h/c，即类HttpRequestParser(HTTP请求解析类)和类HttpResponseParser(Http响应解析结构体)。这两个结构都有成员http_parser。

http_parser是一个用C编写的HTTP消息解析器，可以解析请求和响应。

然后这两个结构分别对应有HttpRequest和HttpResponse。

这两个结构主要含有的函数是execute，即解析协议。观察函数test_request和test_response。

```c++
void test_request(const char *str) {
    sylar::http::HttpRequestParser parser;
    std::string tmp = str;
    std::cout << "<test_request>:" << std::endl
              << tmp << std::endl;
    parser.execute(&tmp[0], tmp.size()); //解析协议
    if (parser.hasError()) {
        std::cout << "parser execute fail" << std::endl;
    } else {
        sylar::http::HttpRequest::ptr req = parser.getData();
        std::cout << req->toString() << std::endl;
    }
}
void test_response(const char *str) {
    sylar::http::HttpResponseParser parser;
    std::string tmp = str;
    std::cout << "<test_response>:" << std::endl
              << tmp << std::endl;
    parser.execute(&tmp[0], tmp.size());
    if (parser.hasError()) {
        std::cout << "parser execue fail" << std::endl;
    } else {
        sylar::http::HttpResponse::ptr rsp = parser.getData();
        std::cout << rsp->toString() << std::endl;
    }
}
```

可以看到，其实主要也是函数execute。

观察函数execute。

```c++
size_t HttpResponseParser::execute(char *data, size_t len) {
    size_t nparsed = http_parser_execute(&m_parser, &s_response_settings, data, len);
    if (m_parser.http_errno != 0) {
        SYLAR_LOG_DEBUG(g_logger) << "parse response fail: " << http_errno_name(HTTP_PARSER_ERRNO(&m_parser));
        setError((int8_t)m_parser.http_errno);
    } else {
        if (nparsed < len) {
            memmove(data, data + nparsed, (len - nparsed));
        }
    }
    return nparsed;
}
```

可以看到主要的函数是http_parser_execute。

http_parser_execute函数很复杂，就不再深入去看了。有一个问题是自己不知道HttpResponseParser中的HttpResponse是如何改变的，execute函数仅仅是对HTTP响应解析器http_parser进行操作的，自己不知道这之中的变化是如何实现的。

其他就没什么了。

可见http_parse.h/c文件和http.h/c文件还是比较孤立的，没有涉及到其他很多联动，或者说继承这些。



### test_http_connection.cc

这里可能就涉及到一些关于调度的一些东西了。

```c++
int main(int argc, char **argv) {
    sylar::IOManager iom(2);
    iom.schedule(run);
    return 0;
}
```

可以看到用到了IOManager。我们直接看run函数。

```c++
void run() {
    sylar::Address::ptr addr = sylar::Address::LookupAnyIPAddress("www.midlane.top:80");
    if (!addr) {
        SYLAR_LOG_INFO(g_logger) << "get addr error";
        return;
    }
    sylar::Socket::ptr sock = sylar::Socket::CreateTCP(addr);
    bool rt                 = sock->connect(addr);
    if (!rt) {
        SYLAR_LOG_INFO(g_logger) << "connect " << *addr << " failed";
        return;
    }
    sylar::http::HttpConnection::ptr conn(new sylar::http::HttpConnection(sock));
    sylar::http::HttpRequest::ptr req(new sylar::http::HttpRequest);
    req->setPath("/");
    req->setHeader("host", "www.midlane.top");
    // 小bug，如果设置了keep-alive，那么要在使用前先调用一次init
    req->setHeader("connection", "keep-alive");
    req->init();
    std::cout << "req:" << std::endl << *req << std::endl;

    conn->sendRequest(req);
    auto rsp = conn->recvResponse();

    if (!rsp) {
        SYLAR_LOG_INFO(g_logger) << "recv response error";
        return;
    }
    std::cout << "rsp:" << std::endl << *rsp << std::endl;

    std::cout << "=========================" << std::endl;

    auto r = sylar::http::HttpConnection::DoGet("http://www.midlane.top/wiki/", 300);
    std::cout << "result=" << r->result << " error=" << r->error << " rsp=" << (r->response ? r->response->toString() : "") << std::endl;
    
    std::cout << "=========================" << std::endl;
    test_pool();
}
```

感觉这里是一个很标准的网络连接案例。

```c++
sylar::Address::ptr addr = sylar::Address::LookupAnyIPAddress("www.midlane.top:80");
    if (!addr) {
        SYLAR_LOG_INFO(g_logger) << "get addr error";
        return;
    }
```

这一步感觉是必须要做的，通过host地址返回对应条件的任意IPAddress，没有地址何谈连接。

```c++
sylar::Socket::ptr sock = sylar::Socket::CreateTCP(addr);
    bool rt                 = sock->connect(addr);
    if (!rt) {
        SYLAR_LOG_INFO(g_logger) << "connect " << *addr << " failed";
        return;
    }
```

之后就是构造socket并进行连接。构造的时候需要addr的协议簇。

```c++
sylar::http::HttpConnection::ptr conn(new sylar::http::HttpConnection(sock));
sylar::http::HttpRequest::ptr req(new sylar::http::HttpRequest);
req->setPath("/");
req->setHeader("host", "www.midlane.top");
// 小bug，如果设置了keep-alive，那么要在使用前先调用一次init
req->setHeader("connection", "keep-alive");
req->init();
std::cout << "req:" << std::endl << *req << std::endl;
```

这里就是初始化HttpConnection(HTTP客户端类)类和HttpRequest(HTTP请求结构)。

首先来看HTTP客户端类。HttpConnection继承SocketStream，SocketStream又继承Stream。

Stream(其实只有SocketStream来继承)表示一种流结构，观察其中的成员函数。

```c++
virtual int read(void* buffer, size_t length) = 0;
virtual int read(ByteArray::ptr ba, size_t length) = 0;
virtual int write(const void* buffer, size_t length) = 0;
virtual int write(ByteArray::ptr ba, size_t length) = 0;
virtual void close() = 0;
virtual int readFixSize(void* buffer, size_t length);
virtual int readFixSize(ByteArray::ptr ba, size_t length);
virtual int writeFixSize(const void* buffer, size_t length);
virtual int writeFixSize(ByteArray::ptr ba, size_t length);
```

其中前五个为纯虚函数，后四个为虚函数。

因为Stream表示为流结构，所以这里处理的也是输入输出，关于从哪里输入输出，则可以衍生出其他很多类，通过继承stream的方式规范化表达。

关于readFixSize/writeFixSize，其实是对read/write的一种封装，即保证读到length长度(因为read/write可能读不到的)。

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
```

观察函数结构即可发现。

之后是SocketStream(socket流)，观察可以发现一共有两个类继承SocketStream，分别是HttpSession(HTTPSession封装)和HttpConnection(HTTP客户端类)。

可以看到，SocketStream有成员类Socket::ptr m_socket;观察重写的read函数：

```c++
int SocketStream::read(void* buffer, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    return m_socket->recv(buffer, length);
}
```

可以看到，这里利用了socket与read的结合，即适应socket应用场景的read函数。

这里临时补充些关于继承/虚函数的小知识。

> 纯虚函数后面需要加=0
>
> override关键字表示：如果派生类在虚函数声明时使用了override描述符，那么该函数必须重载其基类中的同名函数。

之后是类HttpConnection(HTTP客户端类)。观察成员函数可以发现，要不是发送Http请求，就是接收Http响应。可见主要是应用于通信。

至于类HttpRequest(HTTP请求结构)，之前说过就不提了。

```c++
conn->sendRequest(req);
auto rsp = conn->recvResponse();
```

这里其实就是发送HTTP请求，然后再接收HTTP响应。

关于sendRequest函数：

```c++
int HttpConnection::sendRequest(HttpRequest::ptr rsp) {
    std::stringstream ss;
    ss << *rsp;
    std::string data = ss.str();
    return writeFixSize(data.c_str(), data.size());
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
int SocketStream::write(const void* buffer, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    return m_socket->send(buffer, length);
}
```

这里先当于把rsp写入输出流，然后再转化为string格式，然后再转化为一个const char*指针，进入write函数中，由socket进行发送。其实发送的就是HttpRequest(HTTP请求结构)中的内容。

查看data中的内容：

![image-20240709101203982](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240709101203982.png)

recvResponse函数接收Http响应。

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
int SocketStream::read(void* buffer, size_t length) {
    if(!isConnected()) {
        return -1;
    }
    return m_socket->recv(buffer, length);
}
```

这里也是一批一批的读(因为buff_size存在上限)。

查看data即可得到：

![image-20240709101610295](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240709101610295.png)

所以是相当于本地的socket发送http请求，然后服务器立刻回复了。

观察后续的输出：

```c++
if (!rsp) {
        SYLAR_LOG_INFO(g_logger) << "recv response error";
        return;
    }
    std::cout << "rsp:" << std::endl
              << *rsp << std::endl;

    std::cout << "=========================" << std::endl;
```

![image-20240709101941307](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240709101941307.png)

可以看到是相互对应的。

接着看：

```c++
auto r = sylar::http::HttpConnection::DoGet("http://www.midlane.top/wiki/", 300);
    std::cout << "result=" << r->result
              << " error=" << r->error
              << " rsp=" << (r->response ? r->response->toString() : "")
              << std::endl;
```

这里涉及到了函数DoGet。

关于DoGet和sendRequest，感觉后者只是建立连接，前者是进行沟通，拉取内容这样。

```c++
HttpResult::ptr HttpConnection::DoGet(const std::string& url
, uint64_t timeout_ms, const std::map<std::string, std::string>& headers, const std::string& body) {
    Uri::ptr uri = Uri::Create(url);
    if(!uri) {
        return std::make_shared<HttpResult>((int)HttpResult::Error::INVALID_URL
                , nullptr, "invalid url: " + url);
    }
    return DoGet(uri, timeout_ms, headers, body);
}
```

返回值结构为HttpResult。感觉这个就是为了承接返回值而生的类。

成员函数：

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

感觉主要就是HTTP响应结构体以及标记是否错误的错误码和错误描述。



这里就涉及到了类Uri，用来表示URL的。

URL其实就是通俗讲的网址，比如http://baidu.com.

Uri类中的变量为：

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

这其实和URL的格式是相对应的，下面是URL的格式：

```c++
     foo://user@sylar.com:8042/over/there?name=ferret#nose
       \_/   \______________/\_________/ \_________/ \__/
        |           |            |            |        |
     scheme     authority       path        query   fragment
```

除此之外，Uri中主要的函数为Create，其他都是一些getXXX或者setXXX函数。我们来看这个函数：

```c++
Uri::ptr uri = Uri::Create(url);

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

宏观来看这个函数其实就是字符串处理，将字符串赋值给Uri。观察字符串urlstr和生成的uri，可以发现这一特点。

![image-20240709145459194](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240709145459194.png)

另外可以发现其实有两个DoGet函数，区别为一个参数是string，另一个参数为uri。当然第一种也是将string转化为uri继续进行的。

```c++
HttpResult::ptr HttpConnection::DoPost(Uri::ptr uri
                            , uint64_t timeout_ms
                            , const std::map<std::string, std::string>& headers
                            , const std::string& body) {
    return DoRequest(HttpMethod::POST, uri, timeout_ms, headers, body);
}
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
```

这个函数其实是将uri转化为了HttpRequest(http请求报文)，然后还要嵌套一层...

```C++
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

ok这里来真的了。首先还原了url对应的IP地址以及端口，创造了socket，然后发起连接，发送请求报文并接收响应报文。

最终的返回的是HttpResult，即将响应报文和是否发送成功信号进行绑定即可。

可以看到后面的步骤其实和前面的是一致的，区别就在上一个是和IP地址相连，这个是和url相连。

当然还有第三种形式：

```c++
test_pool();
void test_pool() {
    sylar::http::HttpConnectionPool::ptr pool(new sylar::http::HttpConnectionPool(
        "www.midlane.top", "", 80, 10, 1000 * 30, 5));

    sylar::IOManager::GetThis()->addTimer(
        1000, [pool]() {
            auto r = pool->doGet("/", 300);
            std::cout << r->toString() << std::endl;
        },
        true);
}
```

这里涉及到了一个新的类HttpConnectionPool(HTTP请求池)。

成员函数：

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

不过这个和HttpConnection结构差距有点大mmm，看一下后面的操作。

构造函数：

```c++
new sylar::http::HttpConnectionPool("www.midlane.top", "", 80, 10, 1000 * 30, 5)
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

就是给成员变量添加初始值。

后面其实就一个添加定时器的操作了。

```c++
sylar::IOManager::GetThis()->addTimer(
        1000, [pool]() {
            auto r = pool->doGet("/", 300);
            std::cout << r->toString() << std::endl;
        },
        true);
Timer::ptr TimerManager::addTimer(uint64_t ms, std::function<void()> cb,bool recurring) {
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

但是有个问题就是自己可以看出来参数是对应的，但返回值却是不对应的，有点难搞。

首先看回调函数吧。

```c++
[pool]() {
            auto r = pool->doGet("/", 300);
            std::cout << r->toString() << std::endl;
        },
```

关于这里doGet中第一个参数为"/"，其实path路径表示的是host之后的部分。比如我们以第二个样例为例，url为http://www.midlane.top/wiki/，然后我们查看uri的path：

![image-20240709170230285](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240709170230285.png)

可以发现path为host之后的部分，所以这里的"/"，其实也是host之后的部分，整体应该为www.midlane.top/。

剩下部分比自己想象中要麻烦，因为一开始其实是初始化了两个线程。

这部分模块就先这样吧，去除IO操作之后的http自己是明白了的，但关于IO操作的部分还是不太明白，不过自己还是很知足了。


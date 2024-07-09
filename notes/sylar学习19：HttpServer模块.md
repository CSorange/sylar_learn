# sylar学习19：HttpServer模块

HttpServer模块概述

>封装`HttpSession`接收请求报文，发送响应报文，该类继承自`SocketStream`。
>
>封装`Servlet`,虚拟接口，当`Server`命中某个`uri`时，执行对应的`Servlet`。
>
>封装`HttpServer`服务器，继承自`TcpServer`。



## *class* HttpSession

HttpSession（构造函数）

```c++
HttpSession::HttpSession(Socket::ptr sock, bool owner)
    : SocketStream(sock, owner) {
}
```

recvRequest（接收请求报文）

```c++
HttpRequest::ptr HttpSession::recvRequest() {
    HttpRequestParser::ptr parser(new HttpRequestParser);// 创建HttpRequestParser
    uint64_t buff_size = HttpRequestParser::GetHttpRequestBufferSize();// 获取缓冲区大小
    std::shared_ptr<char> buffer(// 交给智能指针托管
        new char[buff_size], [](char *ptr) {
            delete[] ptr;
        });
    char *data = buffer.get();
    int offset = 0;
    do {// 在offset后面接着读数据
        int len = read(data + offset, buff_size - offset);
        if (len <= 0) {
            close();
            return nullptr;
        }
        len += offset;// 当前已经读取的数据长度
        size_t nparse = parser->execute(data, len);// 解析缓冲区data中的数据，execute会将data向前移动nparse个字节，nparse为已经成功解析的字节数
        if (parser->hasError()) {
            close();
            return nullptr;
        }
        offset = len - nparse;// 此时data还剩下已经读到的数据 - 解析过的数据
        if (offset == (int)buff_size) {// 缓冲区满了还没解析完
            close();
            return nullptr;
        }
        if (parser->isFinished()) {// 解析结束
            break;
        }
    } while (true);
    
    parser->getData()->init();
    return parser->getData();
}
```

sendResponse（发送响应报文）

```c++
int HttpSession::sendResponse(HttpResponse::ptr rsp) {
    std::stringstream ss;
    ss << *rsp;
    std::string data = ss.str();
    return writeFixSize(data.c_str(), data.size());
}
```



## *class* Servlet

> 抽象类，基类，提供一个纯虚函数。当访问到该`servlet`时，执行`handle`方法。

```C++
virtual int32_t handle(sylar::http::HttpRequest::ptr request
, sylar::http::HttpResponse::ptr response, sylar::http::HttpSession::ptr session) = 0;
```

成员变量：

```c++
/// 名称
std::string m_name;
```

构造函数：

```C++
Servlet(const std::string& name)
        :m_name(name) {}
```



## *class* FunctionServlet

整体内容：

```c++
class FunctionServlet : public Servlet {
public:
    /// 智能指针类型定义
    typedef std::shared_ptr<FunctionServlet> ptr;
    /// 函数回调类型定义
    typedef std::function<int32_t (sylar::http::HttpRequest::ptr request, sylar::http::HttpResponse::ptr response, sylar::http::HttpSession::ptr session)> callback;
    /**
     * @brief 构造函数
     * @param[in] cb 回调函数
     */
    FunctionServlet(callback cb);
    virtual int32_t handle(sylar::http::HttpRequest::ptr request, sylar::http::HttpResponse::ptr response, sylar::http::HttpSession::ptr session) override;
private:
    /// 回调函数
    callback m_cb;
};
```

handle（执行函数）

```C++
int32_t FunctionServlet::handle(sylar::http::HttpRequest::ptr request, sylar::http::HttpResponse::ptr response, sylar::http::HttpSession::ptr session) {
    return m_cb(request, response, session);
}
```

## *class* ServletDispatch(分发器)

成员变量：

```c++
/// 读写互斥量
RWMutexType m_mutex;
/// 精准匹配servlet MAP
/// uri(/sylar/xxx) -> servlet
std::unordered_map<std::string, IServletCreator::ptr> m_datas;
/// 模糊匹配servlet 数组
/// uri(/sylar/*) -> servlet
std::vector<std::pair<std::string, IServletCreator::ptr> > m_globs;
/// 默认servlet，所有路径都没匹配到时使用
Servlet::ptr m_default;
```

构造函数：

```c++
ServletDispatch::ServletDispatch()
    :Servlet("ServletDispatch") {
    m_default.reset(new NotFoundServlet("sylar/1.0"));
}
```

handle（执行函数）

```c++
int32_t ServletDispatch::handle(sylar::http::HttpRequest::ptr request, sylar::http::HttpResponse::ptr response, sylar::http::HttpSession::ptr session) {
    auto slt = getMatchedServlet(request->getPath());
    if(slt) {
        slt->handle(request, response, session);
    }
    return 0;
}
```

addServlet（添加精准Servlet，智能指针）

```c++
void ServletDispatch::addServlet(const std::string& uri, Servlet::ptr slt) {
    RWMutexType::WriteLock lock(m_mutex);
    m_datas[uri] = std::make_shared<HoldServletCreator>(slt);
}
```

addServlet（添加精准Servlet，回调函数）

```c++
void ServletDispatch::addServlet(const std::string& uri,FunctionServlet::callback cb) {
    RWMutexType::WriteLock lock(m_mutex);
    m_datas[uri] = std::make_shared<HoldServletCreator>(
                        std::make_shared<FunctionServlet>(cb));
}
```

可以添加智能指针，可以添加回调函数，想起了之前的协调函数。

addGlobServlet（添加模糊Servlet，智能指针）

```c++
void ServletDispatch::addGlobServlet(const std::string& uri
                                    ,Servlet::ptr slt) {
    RWMutexType::WriteLock lock(m_mutex);
    for(auto it = m_globs.begin();
            it != m_globs.end(); ++it) {// 将原先的删除
        if(it->first == uri) {
            m_globs.erase(it);
            break;
        }
    }
    m_globs.push_back(std::make_pair(uri
                , std::make_shared<HoldServletCreator>(slt)));
}
```

精准Servlet用的是Map，模糊Servlet用的是Vector。

addServlet（添加模糊Servlet，回调函数）

```c++
void ServletDispatch::addGlobServlet(const std::string& uri,FunctionServlet::callback cb) {
    return addGlobServlet(uri, std::make_shared<FunctionServlet>(cb));
}
```

添加时即将回调函数转化为智能指针。

delServlet（删除精准Servlet）

```c++
void ServletDispatch::delServlet(const std::string& uri) {
    RWMutexType::WriteLock lock(m_mutex);
    m_datas.erase(uri);
}
```

delGlobServlet（删除模糊Servlet）

```c++
void ServletDispatch::delGlobServlet(const std::string& uri) {
    RWMutexType::WriteLock lock(m_mutex);
    for(auto it = m_globs.begin();
            it != m_globs.end(); ++it) {
        if(it->first == uri) {
            m_globs.erase(it);
            break;
        }
    }
}
```

getServlet（返回精准Servlet）

```
Servlet::ptr ServletDispatch::getServlet(const std::string& uri) {
    RWMutexType::ReadLock lock(m_mutex);
    auto it = m_datas.find(uri);
    return it == m_datas.end() ? nullptr : it->second->get();
}
```

getGlobServlet（返回模糊Servlet）

```C++
Servlet::ptr ServletDispatch::getGlobServlet(const std::string& uri) {
    RWMutexType::ReadLock lock(m_mutex);
    for(auto it = m_globs.begin();
            it != m_globs.end(); ++it) {
        if(it->first == uri) {
            return it->second->get();
        }
    }
    return nullptr;
}
```

getMatchedServlet（匹配Servlet）

这里不确定是精准的还是模糊的，所以先精准找一下，再模糊找一下。

```c++
Servlet::ptr ServletDispatch::getMatchedServlet(const std::string& uri) {
    RWMutexType::ReadLock lock(m_mutex);
    auto mit = m_datas.find(uri);// 先找精准的Servlet
    if(mit != m_datas.end()) {
        return mit->second->get();
    }
    for(auto it = m_globs.begin();// 再找模糊的Servlet
            it != m_globs.end(); ++it) {
        if(!fnmatch(it->first.c_str(), uri.c_str(), 0)) {// 模糊匹配Servlet
            return it->second->get();
        }
    }
    return m_default;// 都没有返回404
}
```



## *class* NotFoundServlet

默认返回404.

成员变量：

```c++
std::string m_name;// 服务器版本名称
std::string m_content;// body
```

构造函数：

```c++
NotFoundServlet::NotFoundServlet(const std::string& name)
    :Servlet("NotFoundServlet")
    ,m_name(name) {
    m_content = "<html><head><title>404 Not Found"
        "</title></head><body><center><h1>404 Not Found</h1></center>"
        "<hr><center>" + name + "</center></body></html>";

}
```

handle（执行函数）

```c++
int32_t NotFoundServlet::handle(sylar::http::HttpRequest::ptr request, sylar::http::HttpResponse::ptr response, sylar::http::HttpSession::ptr session) {
    response->setStatus(sylar::http::HttpStatus::NOT_FOUND);
    response->setHeader("Server", "sylar/1.0.0");
    response->setHeader("Content-Type", "text/html");
    response->setBody(m_content);
    return 0;
}
```



## *class* HttpServer

成员变量：

```c++
/// 是否支持长连接
bool m_isKeepalive;
/// Servlet分发器
ServletDispatch::ptr m_dispatch;
```

构造函数：

```c++
HttpServer::HttpServer(bool keepalive,sylar::IOManager* worker
,sylar::IOManager* io_worker,sylar::IOManager* accept_worker)
    :TcpServer(io_worker, accept_worker)
    ,m_isKeepalive(keepalive) {
    m_dispatch.reset(new ServletDispatch);

    m_type = "http";
}
```

setName（设置服务器名称）

```c++
void HttpServer::setName(const std::string& v) {
    TcpServer::setName(v);
    // 设置默认servlet名称
    m_dispatch->setDefault(std::make_shared<NotFoundServlet>(v));
}
```

handleClient（与客户端通信）

```c++
void HttpServer::handleClient(Socket::ptr client) {
    SYLAR_LOG_DEBUG(g_logger) << "handleClient " << *client;
    HttpSession::ptr session(new HttpSession(client));
    do {
        auto req = session->recvRequest();// 接收请求报文
        if(!req) {
            SYLAR_LOG_DEBUG(g_logger) << "recv http request fail, errno="
                << errno << " errstr=" << strerror(errno)
                << " cliet:" << *client << " keep_alive=" << m_isKeepalive;
            break;
        }

        HttpResponse::ptr rsp(new HttpResponse(req->getVersion()
                            ,req->isClose() || !m_isKeepalive));// 创建响应报文
        rsp->setHeader("Server", getName());// 设置Server名Head
        m_dispatch->handle(req, rsp, session);// 执行操作
        session->sendResponse(rsp);// 发送响应报文

        if(!m_isKeepalive || req->isClose()) {// 若不支持长连接，关闭
            break;
        }
    } while(true);
    session->close();// 关闭socket
}
```


# sylar学习25.5：关于最终的test

还是单独开一个md文件来进行梳理吧，毕竟不知道后面会写多少。

这里主要就是test_http_server.cc函数，这次好好观察了一下，确实是最终的aylar服务器的样子了，所以我们就从这个测试函数出发来去学习吧。

```c++
sylar::IOManager iom(1, true, "main");
worker.reset(new sylar::IOManager(3, false, "worker"));
```

这里主函数中构造了两个IOManager，一个是主IOmanager，另一个是worker，不过目前自己还不知道是什么意思...不过后面查看的时候确实有四个线程：

![image-20240730163637117](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240730163637117.png)

可以看到工作线程都阻塞在epoll_wait函数那里。

然后就是添加调度任务：`iom.schedule(run);`。

观察这个run函数：

```c++
g_logger->setLevel(sylar::LogLevel::INFO);
    //sylar::http::HttpServer::ptr server(new sylar::http::HttpServer(true, worker.get(), sylar::IOManager::GetThis()));
    sylar::http::HttpServer::ptr server(new sylar::http::HttpServer(true));
    sylar::Address::ptr addr = sylar::Address::LookupAnyIPAddress("0.0.0.0:8020");
    while (!server->bind(addr)) {
        sleep(2);
    }
```

这里初始化了server。

```c++
HttpServer::HttpServer(bool keepalive
               ,sylar::IOManager* worker
               ,sylar::IOManager* io_worker
               ,sylar::IOManager* accept_worker)
    :TcpServer(io_worker, accept_worker)
    ,m_isKeepalive(keepalive) {
    m_dispatch.reset(new ServletDispatch);

    m_type = "http";
    //m_dispatch->addServlet("/_/status", Servlet::ptr(new StatusServlet));
    //m_dispatch->addServlet("/_/config", Servlet::ptr(new ConfigServlet));
}
```

除了传入的参数true，后面的三个IOManager都是默认的，都为GetThis，即为当前IOManager。可以看到这里初始化了参数m_dispatch。

查看ServletDispatch的构造函数：

```c++
ServletDispatch::ServletDispatch()
    :Servlet("ServletDispatch") {
    m_default.reset(new NotFoundServlet("sylar/1.0"));
}
```

这里继承了serlvet，不过这个其实也只是初始化一个名字。

这里初始化其实只初始化了m_default，即默认servlet。

初始化完了之后初始化地址，然后通过bind进行连接。

```c++
sylar::Address::ptr addr = sylar::Address::LookupAnyIPAddress("0.0.0.0:8020");
    while (!server->bind(addr)) {
        sleep(2);
    }
```

之后获取ServletDispatch，并开始进行一系列的操作。

```c++
auto sd = server->getServletDispatch();
    sd->addServlet("/sylar/xx", [](sylar::http::HttpRequest::ptr req, sylar::http::HttpResponse::ptr rsp, sylar::http::HttpSession::ptr session) {
        rsp->setBody(req->toString());
        return 0;
    });
```

首先是addServlet。

```c++
void ServletDispatch::addServlet(const std::string& uri
                        ,FunctionServlet::callback cb) {
    RWMutexType::WriteLock lock(m_mutex);
    m_datas[uri] = std::make_shared<HoldServletCreator>(
                        std::make_shared<FunctionServlet>(cb));
}
```

观察函数addServlet，其实更像是一个向unordered_map进行添加元素，这个元素的key为url，value为这个构造函数，即后面那一堆。

这里的cb是：rsp->setBody(req->toString())，想是沟通HTTP请求结构和HTTP响应结构体。

后面两个addGlobServlet其实同样的结构。

```c++
sd->addGlobServlet("/sylar/*", [](sylar::http::HttpRequest::ptr req, sylar::http::HttpResponse::ptr rsp, sylar::http::HttpSession::ptr session) {
        rsp->setBody("Glob:\r\n" + req->toString());
        return 0;
    });
```

```C++
sd->addGlobServlet("/sylarx/*", [](sylar::http::HttpRequest::ptr req, sylar::http::HttpResponse::ptr rsp, sylar::http::HttpSession::ptr session) {
        rsp->setBody(XX(<html>
        ......
        return 0;
    });
```

其实这函数addServlet和addGlobServlet本身就是一致的：

```c++
void ServletDispatch::addGlobServlet(const std::string& uri
                                ,FunctionServlet::callback cb) {
    return addGlobServlet(uri, std::make_shared<FunctionServlet>(cb));
}
```

然后就开始了：

```C++
server->start();
```




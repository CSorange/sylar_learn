# nginx学习04：TCP连接建立

主要为两个部分，监听套接字初始化函数ngx_http_optimize_servers和nginx整个连接的过程。

### ngx_http_optimize_servers

主要处理nginx服务的监听套接字。

```c++
/**
 * ngx_http_optimize_servers：处理Nginx服务的监听套接字
 * 说明：主要遍历Nginx服务器提供的端口，然后根据每一个IP地址:port这种配置创建一个监听套接字
 * ngx_http_init_listening：初始化监听套接字
 */
static ngx_int_t
ngx_http_optimize_servers(ngx_conf_t *cf, ngx_http_core_main_conf_t *cmcf,
    ngx_array_t *ports)
{
    ngx_uint_t             p, a;
    ngx_http_conf_port_t  *port;
    ngx_http_conf_addr_t  *addr;
 
    if (ports == NULL) {
        return NGX_OK;
    }
 
    /* 根据Nginx配置的端口号进行遍历 */
    port = ports->elts;
    for (p = 0; p < ports->nelts; p++) {
 
        ngx_sort(port[p].addrs.elts, (size_t) port[p].addrs.nelts,
                 sizeof(ngx_http_conf_addr_t), ngx_http_cmp_conf_addrs);
 
        /*
         * check whether all name-based servers have the same
         * configuration as a default server for given address:port
         */
 
        addr = port[p].addrs.elts;
        for (a = 0; a < port[p].addrs.nelts; a++) {
 
            if (addr[a].servers.nelts > 1
#if (NGX_PCRE)
                || addr[a].default_server->captures
#endif
               )
            {
                if (ngx_http_server_names(cf, cmcf, &addr[a]) != NGX_OK) {
                    return NGX_ERROR;
                }
            }
        }
 
        /* 初始化监听套接字 */
        if (ngx_http_init_listening(cf, &port[p]) != NGX_OK) {
            return NGX_ERROR;
        }
    }
 
    return NGX_OK;
}
```

可以看到这个和自己之前的想法是一致的，就是一个套接字是可以进行转变的，一开始是监听连接，之后就变成了监听read事件。

这里初始化是遍历nginx提供的端口，然后每个IP地址:port对都创建一个监听套接字。其中主要函数为ngx_http_init_listening。

```c++
/**
 * 初始化侦听套接字
 */
static ngx_int_t
ngx_http_init_listening(ngx_conf_t *cf, ngx_http_conf_port_t *port)
{
    ngx_uint_t                 i, last, bind_wildcard;
    ngx_listening_t           *ls;
    ngx_http_port_t           *hport;
    ngx_http_conf_addr_t      *addr;
 
    addr = port->addrs.elts;
    last = port->addrs.nelts;
 
    /*
     * If there is a binding to an "*:port" then we need to bind() to
     * the "*:port" only and ignore other implicit bindings.  The bindings
     * have been already sorted: explicit bindings are on the start, then
     * implicit bindings go, and wildcard binding is in the end.
     */
 
    if (addr[last - 1].opt.wildcard) {
        addr[last - 1].opt.bind = 1;
        bind_wildcard = 1;
 
    } else {
        bind_wildcard = 0;
    }
 
    i = 0;
 
    /* 根据IP地址 遍历 创建 listening*/
    while (i < last) {
 
        if (bind_wildcard && !addr[i].opt.bind) {
            i++;
            continue;
        }
 
        /* 创建侦听套接字 listening */
        ls = ngx_http_add_listening(cf, &addr[i]);
        if (ls == NULL) {
            return NGX_ERROR;
        }
 
        hport = ngx_pcalloc(cf->pool, sizeof(ngx_http_port_t));
        if (hport == NULL) {
            return NGX_ERROR;
        }
 
        ls->servers = hport;
 
        hport->naddrs = i + 1;
 
        switch (ls->sockaddr->sa_family) {
 
#if (NGX_HAVE_INET6)
        case AF_INET6:
            if (ngx_http_add_addrs6(cf, hport, addr) != NGX_OK) {
                return NGX_ERROR;
            }
            break;
#endif
        default: /* AF_INET */
            if (ngx_http_add_addrs(cf, hport, addr) != NGX_OK) {
                return NGX_ERROR;
            }
            break;
        }
 
        if (ngx_clone_listening(cf, ls) != NGX_OK) {
            return NGX_ERROR;
        }
 
        addr++;
        last--;
    }
 
    return NGX_OK;
}
```

这里最重要的函数为ngx_http_add_listening。

```C++
/**
 * 创建侦听套接字 listening
 */
static ngx_listening_t *
ngx_http_add_listening(ngx_conf_t *cf, ngx_http_conf_addr_t *addr)
{
    ngx_listening_t           *ls;
    ngx_http_core_loc_conf_t  *clcf;
    ngx_http_core_srv_conf_t  *cscf;
 
    /* 创建一个套接字 */
    ls = ngx_create_listening(cf, &addr->opt.sockaddr.sockaddr,
                              addr->opt.socklen);
    if (ls == NULL) {
        return NULL;
    }
 
    ls->addr_ntop = 1;
 
    /* 侦听套接字 的回调函数。该回调函数在ngx_event_accept函数中回调；
     * 回调之后，会将读取事件回调函数rev->handler()修改成方法：ngx_http_wait_request_handler*/
    ls->handler = ngx_http_init_connection;
 
    cscf = addr->default_server;
    ls->pool_size = cscf->connection_pool_size;
    ls->post_accept_timeout = cscf->client_header_timeout;
 
    clcf = cscf->ctx->loc_conf[ngx_http_core_module.ctx_index];
 
    ls->logp = clcf->error_log;
    ls->log.data = &ls->addr_text;
    ls->log.handler = ngx_accept_log_error;
 
 
    return ls;
}
```



### nginx整个连接的过程

后面的部分其实在之前event中有涉及过，这里相当于是加上http初始化，监听套接字初始化之后整个过程。

>1.套接字配置
>
>在Nginx main函数的**ngx_init_cycle()**方法中，调用了**ngx_open_listening_sockets函数**，这个函数负责将创建的监听套接字进行套接字选项的设置（比如非阻塞、接受发送的缓冲区、绑定、监听处理）
>
>2.HTTP初始化/套接字创建&初始化
>
>HTTP模块初始化优先于Event模块，HTTP模块通过ngx_http_block()方法进行初始化，然后调用ngx_http_optimize_servers()进行套接字的创建和初始化（ngx_http_init_listening、ngx_http_add_listening、ngx_create_listening）。根据每一个IP地址:port这种配置创建监听套接字。
>
>3.进行监听/初始化客户端连接(这里只是设置了回调函数，还没有运行)
>
>ngx_http_add_listening函数，还会将ls->handler监听套接字的回调函数设置为**ngx_http_init_connection**。ngx_http_init_connection此函数主要初始化一个客户端连接connection。
>
>4.event模块初始化
>
>Event模块的初始化主要调用**ngx_event_process_init()**函数。该函数每个worker工作进程都会初始化调用。然后设置read/write的回调函数。
>
>后面其实就是前面介绍的那些，就不再重复写了。
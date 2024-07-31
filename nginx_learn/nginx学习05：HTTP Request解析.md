# nginx学习05：HTTP Request解析

这里其实相当于已经收到消息了，准备着手开始处理了，即在**ngx_http_wait_request_handler**方法内部。

但自己对于这两个模块之间的联系，之前好像也不是很明白。单独都很明白，但合在一起就不太懂了。

ok下面看一下Request的解析过程。

![img](https://i-blog.csdnimg.cn/blog_migrate/16bee93aa3ecebff761569f57fafd5da.png)

这个图...有点乱感觉。

说明：

> Nginx的HTTP核心模块只解析request的请求行和请求头，不会主动读取HTTP 请求body数据，但是提供了**ngx_http_read_client_request_body**方法，供各个filter模块处理。
>
> ngx_http_wait_request_handler：等待read事件上来，并且等到HTTP的request数据。
>
> ngx_http_process_request_line：处理HTTP的request的请求行。
>
> ngx_http_process_request_header：处理HTTP的request的请求头

这里说的body数据不是自己理解的实际传输的那些数据。

真正处理是在函数ngx_http_process_request中。

```c++
c->read->handler = ngx_http_request_handler;
c->write->handler = ngx_http_request_handler;
```

查看ngx_http_request_handler：

```C++
/**
 * read和write事件都设置为：ngx_http_request_handler，通过事件状态来判断
 */
static void
ngx_http_request_handler(ngx_event_t *ev)
{
    ngx_connection_t    *c;
    ngx_http_request_t  *r;
 
    c = ev->data;
    r = c->data;
 
    ngx_http_set_log_request(c->log, r);
 
    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http run request: \"%V?%V\"", &r->uri, &r->args);
 
    if (ev->write) {
        r->write_event_handler(r);
 
    } else {
        r->read_event_handler(r);
    }
 
    /* 主要用于处理subrequest 子请求 */
    ngx_http_run_posted_requests(c);
}
```



#### 核心分发函数ngx_http_handler

ngx_http_handler函数主要用于设置write事件回调函数。**r->write_event_handler = ngx_http_core_run_phases**

```c++
/**
 * 核心分发函数，主要用于设置write写事件回调函数ngx_http_core_run_phases
 */
void
ngx_http_handler(ngx_http_request_t *r)
{
    ngx_http_core_main_conf_t  *cmcf;
 
    r->connection->log->action = NULL;
 
    if (!r->internal) {
        switch (r->headers_in.connection_type) {
        case 0:
            r->keepalive = (r->http_version > NGX_HTTP_VERSION_10);
            break;
 
        case NGX_HTTP_CONNECTION_CLOSE:
            r->keepalive = 0;
            break;
 
        case NGX_HTTP_CONNECTION_KEEP_ALIVE:
            r->keepalive = 1;
            break;
        }
 
        r->lingering_close = (r->headers_in.content_length_n > 0
                              || r->headers_in.chunked);
        r->phase_handler = 0;
 
    } else {
        cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
        r->phase_handler = cmcf->phase_engine.server_rewrite_index;
    }
 
    r->valid_location = 1;
#if (NGX_HTTP_GZIP)
    r->gzip_tested = 0;
    r->gzip_ok = 0;
    r->gzip_vary = 0;
#endif
 
    /* 设置write事件回调函数，并且执行ngx_http_core_run_phases回调函数 */
    r->write_event_handler = ngx_http_core_run_phases;
    ngx_http_core_run_phases(r);
}
```


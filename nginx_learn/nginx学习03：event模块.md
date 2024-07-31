# nginx学习03：event模块

这里学习一下TCP连接和读取事件逻辑。

>在 Nginx 的初始化启动过程中，worker 工作进程会调用事件模块的ngx_event_process_init 方法为每个监听套接字ngx_listening_t 分配一个 ngx_connection_t 连接，并设置该连接上读事件的回调方法handler 为ngx_event_accept，同时将读事件挂载到epoll 事件机制中等待监听套接字连接上的可读事件发生，到此，Nginx 就可以接收并处理来自客户端的请求。
>

因为肯定是先有连接，然后再有IO操作，所以这里是先处理的连接。

>当监听套接字连接上的可读事件发生时，即该连接上有来自客户端发出的连接请求，则会启动读read事件的handler 回调方法ngx_event_accept，在ngx_event_accept 方法中调用accept() 函数接收来自客户端的连接请求，成功建立连接之后，ngx_event_accept 函数调用监听套接字ngx_listen_t上的handler 回调方法ls->handler(c)（该回调方法就是ngx_http_init_connection 主要用于初始化ngx_connection_t客户端连接）。
>
>ngx_http_init_connection 会将rev->handler的回调函数修改成： ngx_http_wait_request_handler，该回调函数主要用于处理read事件的数据读取。后续当有read事件上来的时候，就会回调ngx_http_wait_request_handler函数，而非ngx_event_accept函数。
>

进行连接之后，就要处理read事件的数据读取，此时的回调函数就会改变，因为一开始处理的是连接，这里处理的是IO，还是有点差别的。

下面是具体的流程：

> 1.ngx_event_process_init 初始化事件循环
>
> 此时rev->handler的回调函数是ngx_event_accept。所以当第一次连接上来，进入事件循环的时候，读取事件调用的是**ngx_event_accept**方法。ngx_event_accept中会调用ls->handler回调函数ngx_http_init_connection。
>
> 2.ngx_http_init_connection 初始化http连接，读取read事件数据
>
> ngx_http_init_connection初始化一个http的连接，并且将rev->handler回调函数修改成ngx_http_wait_request_handler，主要用于接收read事件数据。所有后面进入事件循环后，read事件调用的是ngx_http_wait_request_handler函数。
>
> 3.ls->handler的回调函数是如何赋值的
>
> 你需要看一下ngx_http.c中的ngx_http_block函数，该函数在模块命令初始化的时候会回调，并且调用  ngx_http_optimize_servers方法，并且进一步调用ngx_http_init_listening初始化的监听器，然后最终调用ngx_http_add_listening方法。
>

所以感觉不是分裂出线程然后进行连接的操作，而是一个监听套接字很连贯的动作。

这里需要注意的是，事件和连接是分开的，是两个数据结构，事件对应的rev，连接对应的ls，一个监听套接字对应一个连接，但只有和事件对应上时才会调用ls->handler，即ngx_http_init_connection，然后进行初始化。

## epoll模块

nginx支持多种事件模块，但自己了解的epoll最多。

关于epoll的数据结构和初始化就不看了...

这里的核心函数为**ngx_epoll_process_events**(实现了收集、分发事件接口)和**ngx_epoll_add_event**(添加一个事件)。

```c++
/**
 * 处理事件  ==> ngx_process_events
 * 实现了收集、分发事件接口(void) ngx_process_events(cycle, timer, flags);
 */
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    int                events;
    uint32_t           revents;
    ngx_int_t          instance, i;
    ngx_uint_t         level;
    ngx_err_t          err;
    ngx_event_t       *rev, *wev;
    ngx_queue_t       *queue;
    ngx_connection_t  *c;
 
    /* NGX_TIMER_INFINITE == INFTIM */
 
    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "epoll timer: %M", timer);
 
    /* 调用epoll_wait获取事件   */
    events = epoll_wait(ep, event_list, (int) nevents, timer);
 
    err = (events == -1) ? ngx_errno : 0;
 
    /* 更新时间 */
    if (flags & NGX_UPDATE_TIME || ngx_event_timer_alarm) {
        ngx_time_update();
    }
 
    /* epoll_wait出错处理  */
    if (err) {
        if (err == NGX_EINTR) {
 
            if (ngx_event_timer_alarm) {
                ngx_event_timer_alarm = 0;
                return NGX_OK;
            }
 
            level = NGX_LOG_INFO;
 
        } else {
            level = NGX_LOG_ALERT;
        }
 
        ngx_log_error(level, cycle->log, err, "epoll_wait() failed");
        return NGX_ERROR;
    }
 
    /* 本次调用没有事件发生  */
    if (events == 0) {
        if (timer != NGX_TIMER_INFINITE) {
            return NGX_OK;
        }
 
        ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                      "epoll_wait() returned no events without timeout");
        return NGX_ERROR;
    }
 
    /* 遍历本次epoll_wait返回的所有事件  */
    for (i = 0; i < events; i++) {
    	/* 获取连接ngx_connection_t的地址  */
        c = event_list[i].data.ptr;
 
        instance = (uintptr_t) c & 1;
        c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);
 
        /* 取出读事件  */
        rev = c->read;
 
        if (c->fd == -1 || rev->instance != instance) {
 
            /*
             * the stale event from a file descriptor
             * that was just closed in this iteration
             */
 
            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll: stale event %p", c);
            continue;
        }
 
        /* 取出事件类型  */
        revents = event_list[i].events;
 
        ngx_log_debug3(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "epoll: fd:%d ev:%04XD d:%p",
                       c->fd, revents, event_list[i].data.ptr);
 
        if (revents & (EPOLLERR|EPOLLHUP)) {
            ngx_log_debug2(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll_wait() error on fd:%d ev:%04XD",
                           c->fd, revents);
        }
 
#if 0
        if (revents & ~(EPOLLIN|EPOLLOUT|EPOLLERR|EPOLLHUP)) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                          "strange epoll_wait() events fd:%d ev:%04XD",
                          c->fd, revents);
        }
#endif
 
        if ((revents & (EPOLLERR|EPOLLHUP))
             && (revents & (EPOLLIN|EPOLLOUT)) == 0)
        {
            /*
             * if the error events were returned without EPOLLIN or EPOLLOUT,
             * then add these flags to handle the events at least in one
             * active handler
             */
 
            revents |= EPOLLIN|EPOLLOUT;
        }
 
        /* 读取事件 EPOLLIN */
        if ((revents & EPOLLIN) && rev->active) {
 
#if (NGX_HAVE_EPOLLRDHUP)
            if (revents & EPOLLRDHUP) {
                rev->pending_eof = 1;
            }
 
            rev->available = 1;
#endif
 
            rev->ready = 1;
 
            /* 如果事件抢到锁，则放入事件队列 */
            if (flags & NGX_POST_EVENTS) {
                queue = rev->accept ? &ngx_posted_accept_events
                                    : &ngx_posted_events;
 
                ngx_post_event(rev, queue);
 
            } else {
            	/* 没有抢到锁，立即调用事件回调方法来处理这个事件  */
                rev->handler(rev);
            }
        }
 
        wev = c->write;
 
        /* 写事件 EPOLLOUT*/
        if ((revents & EPOLLOUT) && wev->active) {
 
            if (c->fd == -1 || wev->instance != instance) {
 
                /*
                 * the stale event from a file descriptor
                 * that was just closed in this iteration
                 */
 
                ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                               "epoll: stale event %p", c);
                continue;
            }
 
            wev->ready = 1;
#if (NGX_THREADS)
            wev->complete = 1;
#endif
 
            if (flags & NGX_POST_EVENTS) {
                ngx_post_event(wev, &ngx_posted_events);
 
            } else {
                wev->handler(wev);
            }
        }
    }
 
    return NGX_OK;
}
```

这个函数其实和自己的idle函数很像了，都是epoll_wait，然后接收到事件之后收集，然后进行分发。

这里说一点：

```c++
if (flags & NGX_POST_EVENTS) {
                ngx_post_event(wev, &ngx_posted_events);
 
            } else {
                wev->handler(wev);
            }
```

这里如果如果进程抢到锁，则放入事件队列，如果没有，则直接处理read事件。

这里自己是这样理解的，如果抢到锁了，就将accept或者read事件放入到队列中延后处理，本着先处理accept，在处理read的原则。但如果抢到锁了，那就一定是read操作，直接处理就ok。

**next one：**

```c++
/**
 * 添加一个事件
 */
static ngx_int_t
ngx_epoll_add_event(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags)
{
    int                  op;
    uint32_t             events, prev;
    ngx_event_t         *e;
    ngx_connection_t    *c;
    struct epoll_event   ee;
 
    /* 每个事件的data成员存放着其对应的ngx_connection_t连接  */
    c = ev->data;
 
    events = (uint32_t) event;
 
    /* 判断事件类型。如果是写事件，c->write 如果是读事件，c->read*/
    if (event == NGX_READ_EVENT) {
        e = c->write;
        prev = EPOLLOUT;
#if (NGX_READ_EVENT != EPOLLIN|EPOLLRDHUP)
        events = EPOLLIN|EPOLLRDHUP;
#endif
 
    } else {
        e = c->read;
        prev = EPOLLIN|EPOLLRDHUP;
#if (NGX_WRITE_EVENT != EPOLLOUT)
        events = EPOLLOUT;
#endif
    }
 
    /* 依据active标志位确定是否为活跃事件  */
    if (e->active) {
        op = EPOLL_CTL_MOD;
        events |= prev;
 
    } else {
        op = EPOLL_CTL_ADD;
    }
 
    ee.events = events | (uint32_t) flags;
    ee.data.ptr = (void *) ((uintptr_t) c | ev->instance);
 
    ngx_log_debug3(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                   "epoll add event: fd:%d op:%d ev:%08XD",
                   c->fd, op, ee.events);
 
    /* 新增一个事件 */
    if (epoll_ctl(ep, op, c->fd, &ee) == -1) {
        ngx_log_error(NGX_LOG_ALERT, ev->log, ngx_errno,
                      "epoll_ctl(%d, %d) failed", op, c->fd);
        return NGX_ERROR;
    }
 
    ev->active = 1;
#if 0
    ev->oneshot = (flags & NGX_ONESHOT_EVENT) ? 1 : 0;
#endif
 
    return NGX_OK;
}
```

这里是添加一个事件，其实就对应着epoll_ctl，注册函数，可以看到函数中主要其实也是这个函数。
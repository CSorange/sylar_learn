# nginx学习02：多进程实现/惊群与负载均衡

默认情况下，nginx为多进程的运行模式，有以下好处：

> 每个进程的资源独立
>
> 不需要添加各种繁琐的锁



#### ngx_master_process_cycle 进入多进程模式

这里主要做的两个工作，主进程进行信号的监听喝处理，并开启子进程。

```c++
/**
 * Nginx的多进程运行模式
 */
void ngx_master_process_cycle(ngx_cycle_t *cycle) {
	char *title;
	u_char *p;
	size_t size;
	ngx_int_t i;
	ngx_uint_t n, sigio;
	sigset_t set;
	struct itimerval itv;
	ngx_uint_t live;
	ngx_msec_t delay;
	ngx_listening_t *ls;
	ngx_core_conf_t *ccf;
 
	/* 设置能接收到的信号 */
	sigemptyset(&set);
	sigaddset(&set, SIGCHLD);
	sigaddset(&set, SIGALRM);
	sigaddset(&set, SIGIO);
	sigaddset(&set, SIGINT);
	sigaddset(&set, ngx_signal_value(NGX_RECONFIGURE_SIGNAL));
	sigaddset(&set, ngx_signal_value(NGX_REOPEN_SIGNAL));
	sigaddset(&set, ngx_signal_value(NGX_NOACCEPT_SIGNAL));
	sigaddset(&set, ngx_signal_value(NGX_TERMINATE_SIGNAL));
	sigaddset(&set, ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
	sigaddset(&set, ngx_signal_value(NGX_CHANGEBIN_SIGNAL));
 
	if (sigprocmask(SIG_BLOCK, &set, NULL) == -1) {
		ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
				"sigprocmask() failed");
	}
 
	sigemptyset(&set);
 
	size = sizeof(master_process);
 
	for (i = 0; i < ngx_argc; i++) {
		size += ngx_strlen(ngx_argv[i]) + 1;
	}
 
	/* 保存进程标题 */
	title = ngx_pnalloc(cycle->pool, size);
	if (title == NULL) {
		/* fatal */
		exit(2);
	}
 
	p = ngx_cpymem(title, master_process, sizeof(master_process) - 1);
	for (i = 0; i < ngx_argc; i++) {
		*p++ = ' ';
		p = ngx_cpystrn(p, (u_char *) ngx_argv[i], size);
	}
 
	ngx_setproctitle(title);
 
	/* 获取核心配置 ngx_core_conf_t */
	ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
 
	/* 启动工作进程 - 多进程启动的核心函数 */
	ngx_start_worker_processes(cycle, ccf->worker_processes,
			NGX_PROCESS_RESPAWN);
	ngx_start_cache_manager_processes(cycle, 0);
 
	ngx_new_binary = 0;
	delay = 0;
	sigio = 0;
	live = 1;
 
	/* 主线程循环 */
	for (;;) {
 
		/* delay用来设置等待worker推出的时间，master接受了退出信号后，
		 * 首先发送退出信号给worker，而worker退出需要一些时间*/
		if (delay) {
			if (ngx_sigalrm) {
				sigio = 0;
				delay *= 2;
				ngx_sigalrm = 0;
			}
 
			ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
					"termination cycle: %M", delay);
 
			itv.it_interval.tv_sec = 0;
			itv.it_interval.tv_usec = 0;
			itv.it_value.tv_sec = delay / 1000;
			itv.it_value.tv_usec = (delay % 1000) * 1000;
 
			if (setitimer(ITIMER_REAL, &itv, NULL) == -1) {
				ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
						"setitimer() failed");
			}
		}
 
		ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "sigsuspend");
 
		/* 等待信号的到来，阻塞函数 */
		sigsuspend(&set);
 
		ngx_time_update();
 
		ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
				"wake up, sigio %i", sigio);
 
		/* 收到了SIGCHLD信号，有worker退出(ngx_reap == 1) */
		if (ngx_reap) {
			ngx_reap = 0;
			ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "reap children");
 
			live = ngx_reap_children(cycle);
		}
 
		if (!live && (ngx_terminate || ngx_quit)) {
			ngx_master_process_exit(cycle);
		}
 
		/* 中止进程  */
		if (ngx_terminate) {
			if (delay == 0) {
				delay = 50;
			}
 
			if (sigio) {
				sigio--;
				continue;
			}
 
			sigio = ccf->worker_processes + 2 /* cache processes */;
 
			if (delay > 1000) {
				ngx_signal_worker_processes(cycle, SIGKILL);
			} else {
				ngx_signal_worker_processes(cycle,
						ngx_signal_value(NGX_TERMINATE_SIGNAL));
			}
 
			continue;
		}
 
		/* 退出进程 */
		if (ngx_quit) {
			ngx_signal_worker_processes(cycle,
					ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
 
			ls = cycle->listening.elts;
			for (n = 0; n < cycle->listening.nelts; n++) {
				if (ngx_close_socket(ls[n].fd) == -1) {
					ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_socket_errno,
							ngx_close_socket_n " %V failed", &ls[n].addr_text);
				}
			}
			cycle->listening.nelts = 0;
 
			continue;
		}
 
		/* 收到SIGHUP信号 重新初始化配置 */
		if (ngx_reconfigure) {
			ngx_reconfigure = 0;
 
			if (ngx_new_binary) {
				ngx_start_worker_processes(cycle, ccf->worker_processes,
						NGX_PROCESS_RESPAWN);
				ngx_start_cache_manager_processes(cycle, 0);
				ngx_noaccepting = 0;
 
				continue;
			}
 
			ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reconfiguring");
 
			cycle = ngx_init_cycle(cycle);
			if (cycle == NULL) {
				cycle = (ngx_cycle_t *) ngx_cycle;
				continue;
			}
 
			ngx_cycle = cycle;
			ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx,
					ngx_core_module);
			ngx_start_worker_processes(cycle, ccf->worker_processes,
					NGX_PROCESS_JUST_RESPAWN);
			ngx_start_cache_manager_processes(cycle, 1);
 
			/* allow new processes to start */
			ngx_msleep(100);
 
			live = 1;
			ngx_signal_worker_processes(cycle,
					ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
		}
 
		/* 当ngx_noaccepting==1时，会把ngx_restart设为1，重启worker  */
		if (ngx_restart) {
			ngx_restart = 0;
			ngx_start_worker_processes(cycle, ccf->worker_processes,
					NGX_PROCESS_RESPAWN);
			ngx_start_cache_manager_processes(cycle, 0);
			live = 1;
		}
 
		/* 收到SIGUSR1信号，重新打开log文件 */
		if (ngx_reopen) {
			ngx_reopen = 0;
			ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
			ngx_reopen_files(cycle, ccf->user);
			ngx_signal_worker_processes(cycle,
					ngx_signal_value(NGX_REOPEN_SIGNAL));
		}
 
		/* SIGUSER2，热代码替换 */
		if (ngx_change_binary) {
			ngx_change_binary = 0;
			ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "changing binary");
			ngx_new_binary = ngx_exec_new_binary(cycle, ngx_argv);
		}
 
		/* 收到SIGWINCH信号不在接受请求，worker退出，master不退出 */
		if (ngx_noaccept) {
			ngx_noaccept = 0;
			ngx_noaccepting = 1;
			ngx_signal_worker_processes(cycle,
					ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
		}
	}
}
```

其中在进入主线程循环之前需要先创建并启动工作进程。

#### **ngx_start_worker_processes**  创建工作进程

这里循环建立了N个子进程，每个子进程都有独立的内存空间。

```c++
/**
 * 创建工作进程
 */
static void ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n,
		ngx_int_t type) {
	ngx_int_t i;
	ngx_channel_t ch;
 
	ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "start worker processes");
 
	ngx_memzero(&ch, sizeof(ngx_channel_t));
 
	ch.command = NGX_CMD_OPEN_CHANNEL;
 
	/* 循环创建工作进程  默认ccf->worker_processes=8个进程，根据CPU个数决定   */
	for (i = 0; i < n; i++) {
 
		/* 打开工作进程  （ngx_worker_process_cycle 回调函数，主要用于处理每个工作线程）*/
		ngx_spawn_process(cycle, ngx_worker_process_cycle,
				(void *) (intptr_t) i, "worker process", type);
 
		ch.pid = ngx_processes[ngx_process_slot].pid;
		ch.slot = ngx_process_slot;
		ch.fd = ngx_processes[ngx_process_slot].channel[0];
 
		ngx_pass_open_channel(cycle, &ch);
	}
}
```

这里最重要的即为`ngx_spawn_process(cycle, ngx_worker_process_cycle, (void *) (intptr_t) i, "worker process", type);`



#### ngx_spawn_process 通过fork得到工作进程

fork相关的代码：

```c++
    /* fork 一个子进程 */
    pid = fork();
 
    switch (pid) {
 
    case -1:
        ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                      "fork() failed while spawning \"%s\"", name);
        ngx_close_channel(ngx_processes[s].channel, cycle->log);
        return NGX_INVALID_PID;
 
    case 0:
    	/* 如果pid fork成功，则调用 ngx_worker_process_cycle方法 */
        ngx_pid = ngx_getpid();
        proc(cycle, data);
        break;
 
    default:
        break;
    }
```

当然，和线程一样，如果创建成功，我们就要调用回调函数。



#### **ngx_worker_process_cycle** 子进程的回调函数

子进程的回调函数，一切子进程的工作从这个方法开始。进入这个函数之后，我们首先要进行初始化，通过`ngx_worker_process_init`进行，当然，因为nginx的进程最终是由时间驱动的，所以最终会调用事件驱动的核心函数`ngx_process_events_and_timers`。

```c++
/**
 * 子进程 回调函数
 * 每个进程的逻辑处理就从这个方法开始
 */
static void ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data) {
	ngx_int_t worker = (intptr_t) data;
 
	ngx_process = NGX_PROCESS_WORKER;
	ngx_worker = worker;
 
	/* 工作进程初始化 */
	ngx_worker_process_init(cycle, worker);
 
	ngx_setproctitle("worker process");
 
	/* 进程循环 */
	for (;;) {
 
		/* 判断是否是退出的状态，如果退出，则需要清空socket连接句柄 */
		if (ngx_exiting) {
			ngx_event_cancel_timers();
 
			if (ngx_event_timer_rbtree.root
					== ngx_event_timer_rbtree.sentinel) {
				ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
 
				ngx_worker_process_exit(cycle);
			}
		}
 
		ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0, "worker cycle");
 
		/* 事件驱动核心函数 */
		ngx_process_events_and_timers(cycle);
 
		if (ngx_terminate) {
			ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
 
			ngx_worker_process_exit(cycle);
		}
 
		/* 如果是退出 */
		if (ngx_quit) {
			ngx_quit = 0;
			ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
					"gracefully shutting down");
			ngx_setproctitle("worker process is shutting down");
 
			if (!ngx_exiting) {
				ngx_exiting = 1;
				ngx_close_listening_sockets(cycle);
				ngx_close_idle_connections(cycle);
			}
		}
 
		/* 如果是重启 */
		if (ngx_reopen) {
			ngx_reopen = 0;
			ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
			ngx_reopen_files(cycle, -1);
		}
	}
}
```



#### ngx_worker_process_init 工作进程初始化

```c++
/**
 * 工作进程初始化
 */
static void ngx_worker_process_init(ngx_cycle_t *cycle, ngx_int_t worker) {
	sigset_t set;
	ngx_int_t n;
	ngx_uint_t i;
	ngx_cpuset_t *cpu_affinity;
	struct rlimit rlmt;
	ngx_core_conf_t *ccf;
	ngx_listening_t *ls;
 
	/* 配置环境变量 */
	if (ngx_set_environment(cycle, NULL) == NULL) {
		/* fatal */
		exit(2);
	}
 
	/* 获取核心配置 */
	ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
 
	if (worker >= 0 && ccf->priority != 0) {
		if (setpriority(PRIO_PROCESS, 0, ccf->priority) == -1) {
			ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
					"setpriority(%d) failed", ccf->priority);
		}
	}
 
	if (ccf->rlimit_nofile != NGX_CONF_UNSET) {
		rlmt.rlim_cur = (rlim_t) ccf->rlimit_nofile;
		rlmt.rlim_max = (rlim_t) ccf->rlimit_nofile;
 
		if (setrlimit(RLIMIT_NOFILE, &rlmt) == -1) {
			ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
					"setrlimit(RLIMIT_NOFILE, %i) failed", ccf->rlimit_nofile);
		}
	}
 
	if (ccf->rlimit_core != NGX_CONF_UNSET) {
		rlmt.rlim_cur = (rlim_t) ccf->rlimit_core;
		rlmt.rlim_max = (rlim_t) ccf->rlimit_core;
 
		if (setrlimit(RLIMIT_CORE, &rlmt) == -1) {
			ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
					"setrlimit(RLIMIT_CORE, %O) failed", ccf->rlimit_core);
		}
	}
 
	/* 设置UID GROUPUID */
	if (geteuid() == 0) {
		if (setgid(ccf->group) == -1) {
			ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
					"setgid(%d) failed", ccf->group);
			/* fatal */
			exit(2);
		}
 
		if (initgroups(ccf->username, ccf->group) == -1) {
			ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
					"initgroups(%s, %d) failed", ccf->username, ccf->group);
		}
 
		if (setuid(ccf->user) == -1) {
			ngx_log_error(NGX_LOG_EMERG, cycle->log, ngx_errno,
					"setuid(%d) failed", ccf->user);
			/* fatal */
			exit(2);
		}
	}
 
	/* 设置CPU亲和性 */
	if (worker >= 0) {
		cpu_affinity = ngx_get_cpu_affinity(worker);
 
		if (cpu_affinity) {
			ngx_setaffinity(cpu_affinity, cycle->log);
		}
	}
 
#if (NGX_HAVE_PR_SET_DUMPABLE)
 
	/* allow coredump after setuid() in Linux 2.4.x */
 
	if (prctl(PR_SET_DUMPABLE, 1, 0, 0, 0) == -1) {
		ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
				"prctl(PR_SET_DUMPABLE) failed");
	}
 
#endif
 
	/* 切换工作目录 */
	if (ccf->working_directory.len) {
		if (chdir((char *) ccf->working_directory.data) == -1) {
			ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
					"chdir(\"%s\") failed", ccf->working_directory.data);
			/* fatal */
			exit(2);
		}
	}
 
	sigemptyset(&set);
 
	/* 清除所有信号 */
	if (sigprocmask(SIG_SETMASK, &set, NULL) == -1) {
		ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
				"sigprocmask() failed");
	}
 
	srandom((ngx_pid << 16) ^ ngx_time());
 
	/*
	 * disable deleting previous events for the listening sockets because
	 * in the worker processes there are no events at all at this point
	 */
	/* 清除sokcet的监听 */
	ls = cycle->listening.elts;
	for (i = 0; i < cycle->listening.nelts; i++) {
		ls[i].previous = NULL;
	}
 
	/* 对模块初始化  */
	for (i = 0; cycle->modules[i]; i++) {
		if (cycle->modules[i]->init_process) {
			if (cycle->modules[i]->init_process(cycle) == NGX_ERROR) {
				/* fatal */
				exit(2);
			}
		}
	}
 
	/**
	 *将其他进程的channel[1]关闭，自己的channel[0]关闭
	 */
	for (n = 0; n < ngx_last_process; n++) {
 
		if (ngx_processes[n].pid == -1) {
			continue;
		}
 
		if (n == ngx_process_slot) {
			continue;
		}
 
		if (ngx_processes[n].channel[1] == -1) {
			continue;
		}
 
		if (close(ngx_processes[n].channel[1]) == -1) {
			ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
					"close() channel failed");
		}
	}
 
	if (close(ngx_processes[ngx_process_slot].channel[0]) == -1) {
		ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
				"close() channel failed");
	}
 
#if 0
	ngx_last_process = 0;
#endif
 
	/**
	 * 给ngx_channel注册一个读事件处理函数
	 */
	if (ngx_add_channel_event(cycle, ngx_channel, NGX_READ_EVENT,
			ngx_channel_handler) == NGX_ERROR) {
		/* fatal */
		exit(2);
	}
}
```



关于sylar多线程的运行，因为开始都是从线程池的构造来的：

```C++
m_threads.resize(m_threadCount);
    for (size_t i = 0; i < m_threadCount; i++) {
        m_threads[i].reset(new Thread(std::bind(&Scheduler::run, this),
                                      m_name + "_" + std::to_string(i)));
        m_threadIds.push_back(m_threads[i]->getId());
    }
```

所以后面就会进入到线程的构造函数中，这里走的就是thread初始化，到run函数中，在这里修改名称等等，然后进入回调函数，这里是Scheduler::run，即调度器的运行函数run中。所以先当于这些子线程都是在运行调度任务的。不过值得一提的是，task中可以设置指定线程，不过sylar中并没有实现，没有进行判断一下之类的。



## 关于惊群

惊群就是说多个进程/线程在等待同一资源的时候，如果资源可用，所有的进程/线程都来去竞争资源的现象。

ps：之前想了一下，感觉很容易或者说简单就可以解决，现在想了一下，没有这么容易hh。

想了一下自己似乎没有实现对于惊群的处理。

**如何解决：**

> Nginx的N个进程会争抢文件锁，当只有拿到**文件锁**的进程，才能处理**accept的事件**。
>
> 没有拿到文件锁的进程，只能处理当前连接对象的**read事件**。

首先通过**ngx_trylock_accept_mutex**争抢文件锁，拿到文件锁的，才可以处理accept事件。

```c++
/**
 * 获取accept锁
 */
ngx_int_t ngx_trylock_accept_mutex(ngx_cycle_t *cycle) {
	/**
	 * 拿到锁
	 */
	if (ngx_shmtx_trylock(&ngx_accept_mutex)) {
 
		ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
				"accept mutex locked");
 
		/* 多次进来，判断是否已经拿到锁 */
		if (ngx_accept_mutex_held && ngx_accept_events == 0) {
			return NGX_OK;
		}
 
		/* 调用ngx_enable_accept_events，开启监听accpet事件*/
		if (ngx_enable_accept_events(cycle) == NGX_ERROR) {
			ngx_shmtx_unlock(&ngx_accept_mutex);
			return NGX_ERROR;
		}
 
		ngx_accept_events = 0;
		ngx_accept_mutex_held = 1;
 
		return NGX_OK;
	}
 
	ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
			"accept mutex lock failed: %ui", ngx_accept_mutex_held);
 
	/**
	 * 没有拿到锁，但是ngx_accept_mutex_held=1
	 */
	if (ngx_accept_mutex_held) {
		/* 没有拿到锁，调用ngx_disable_accept_events，将accpet事件删除 */
		if (ngx_disable_accept_events(cycle, 0) == NGX_ERROR) {
			return NGX_ERROR;
		}
 
		ngx_accept_mutex_held = 0;
	}
 
	return NGX_OK;
}
```

> ngx_accept_mutex_held是拿到锁的一个标志，当拿到锁了，flags会被设置成NGX_POST_EVENTS，这个标志会在事件处理函数ngx_process_events中将所有事件（accept和read）放入对应的ngx_posted_accept_events和ngx_posted_events队列中进行延后处理。
>
> 当没有拿到锁，调用事件处理函数**ngx_process_events**的时候，可以明确都是read的事件，所以可以直接调用事件ev->handler方法回调处理。
>
> 拿到锁的进程，接下来会优先处理ngx_posted_accept_events队列上的accept事件，处理函数：ngx_event_process_posted
>
> 处理完accept事件后，就将文件锁释放
>
> 接下来处理ngx_posted_events队列上的read事件，处理函数：ngx_event_process_posted

即相当于有两个队列，一个是ngx_posted_accept_events，上面都是一些accept事件，另外还有ngx_posted_events，上面都是read事件。

如果有锁的标志，就先处理ngx_posted_accept_events中的accept事件，处理完之后将文件锁释放，然后再处理另一个队列上的read事件。如果没有，就先处理ngx_posted_events上的read事件。

查看ngx_process_events 事件的核心处理函数

```C++
        /* 读取事件 EPOLLIN */
        if ((revents & EPOLLIN) && rev->active) {
 
#if (NGX_HAVE_EPOLLRDHUP)
            if (revents & EPOLLRDHUP) {
                rev->pending_eof = 1;
            }
 
            rev->available = 1;
#endif
 
            rev->ready = 1;
 
            /* 如果进程抢到锁，则放入事件队列 */
            if (flags & NGX_POST_EVENTS) {
                queue = rev->accept ? &ngx_posted_accept_events
                                    : &ngx_posted_events;
 
                ngx_post_event(rev, queue);
 
            } else {
            	/* 没有抢到锁，则直接处理read事件*/
                rev->handler(rev);
            }
        }
```



## 关于负载均衡



当单个进程总的connection连接数达到总数的**7/8**的时候，就不会再接收新的accpet事件。

如果拿到锁的进程能很快处理完accpet，而没拿到锁的一直在等待（等待时延：**ngx_accept_mutex_delay**），容易造成进程忙的很忙，空的很空

这里主要是通过进程事件分发器来进行实现的。

>当事件配置初始化的时候，会设置一个全局变量：ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
>
>当ngx_accept_disabled为正数的时候，connection达到连接总数的7/8的时候，就不再处理新的连接accept事件，只处理当前连接的read事件。

主要在函数ngx_process_events_and_timers中实现。

```c++
/**
 * 进程事件分发器
 */
void ngx_process_events_and_timers(ngx_cycle_t *cycle) {
	ngx_uint_t flags;
	ngx_msec_t timer, delta;
 
	if (ngx_timer_resolution) {
		timer = NGX_TIMER_INFINITE;
		flags = 0;
 
	} else {
		timer = ngx_event_find_timer();
		flags = NGX_UPDATE_TIME;
 
#if (NGX_WIN32)
 
		/* handle signals from master in case of network inactivity */
 
		if (timer == NGX_TIMER_INFINITE || timer > 500) {
			timer = 500;
		}
 
#endif
	}
 
	/**
	 * ngx_use_accept_mutex变量代表是否使用accept互斥体
	 * 默认是使用，可以通过accept_mutex off;指令关闭；
	 * accept mutex 的作用就是避免惊群，同时实现负载均衡
	 */
	if (ngx_use_accept_mutex) {
 
		/**
		 * 	ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
		 * 	当connection达到连接总数的7/8的时候，就不再处理新的连接accept事件，只处理当前连接的read事件
		 * 	这个是比较简单的一种负载均衡方法
		 */
		if (ngx_accept_disabled > 0) {
			ngx_accept_disabled--;
 
		} else {
			/* 获取锁失败 */
			if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
				return;
			}
 
			/* 拿到锁 */
			if (ngx_accept_mutex_held) {
				/**
				 * 给flags增加标记NGX_POST_EVENTS，这个标记作为处理时间核心函数ngx_process_events的一个参数，这个函数中所有事件将延后处理。
				 * accept事件都放到ngx_posted_accept_events链表中，
				 * epollin|epollout普通事件都放到ngx_posted_events链表中
				 **/
				flags |= NGX_POST_EVENTS;
 
			} else {
 
				/**
				 * 1. 获取锁失败，意味着既不能让当前worker进程频繁的试图抢锁，也不能让它经过太长事件再去抢锁
				 * 2. 开启了timer_resolution时间精度，需要让ngx_process_change方法在没有新事件的时候至少等待ngx_accept_mutex_delay毫秒之后再去试图抢锁
				 * 3. 没有开启时间精度时，如果最近一个定时器事件的超时时间距离现在超过了ngx_accept_mutex_delay毫秒，也要把timer设置为ngx_accept_mutex_delay毫秒
				 * 4. 不能让ngx_process_change方法在没有新事件的时候等待的时间超过ngx_accept_mutex_delay，这会影响整个负载均衡机制
				 * 5. 如果拿到锁的进程能很快处理完accpet，而没拿到锁的一直在等待，容易造成进程忙的很忙，空的很空
				 */
				if (timer == NGX_TIMER_INFINITE
						|| timer > ngx_accept_mutex_delay) {
					timer = ngx_accept_mutex_delay;
				}
			}
		}
	}
 
	delta = ngx_current_msec;
 
	/**
	 * 事件调度函数
	 * 1. 当拿到锁，flags=NGX_POST_EVENTS的时候，不会直接处理事件，
	 * 将accept事件放到ngx_posted_accept_events，read事件放到ngx_posted_events队列
	 * 2. 当没有拿到锁，则处理的全部是read事件，直接进行回调函数处理
	 * 参数：timer-epoll_wait超时时间  (ngx_accept_mutex_delay-延迟拿锁事件   NGX_TIMER_INFINITE-正常的epollwait等待事件)
	 */
	(void) ngx_process_events(cycle, timer, flags);
 
	delta = ngx_current_msec - delta;
 
	ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
			"timer delta: %M", delta);
	/**
	 * 1. ngx_posted_accept_events是一个事件队列，暂存epoll从监听套接口wait到的accept事件
	 * 2. 这个方法是循环处理accpet事件列队上的accpet事件
	 */
	ngx_event_process_posted(cycle, &ngx_posted_accept_events);
 
	/**
	 * 如果拿到锁，处理完accept事件后，则释放锁
	 */
	if (ngx_accept_mutex_held) {
		ngx_shmtx_unlock(&ngx_accept_mutex);
	}
 
	if (delta) {
		ngx_event_expire_timers();
	}
 
	/**
	 *1. 普通事件都会存放在ngx_posted_events队列上
	 *2. 这个方法是循环处理read事件列队上的read事件
	 */
	ngx_event_process_posted(cycle, &ngx_posted_events);
}
```

其实主要就是这个函数，要知道惊群和负载均衡对应的是我们epoll_waitf返回之后对任务的进行分发处理。

在sylar中这个部分其实没有nginx这些的处理，都是直接放入shedule进行调度的，但nginx不一样，一个是accept事件的加锁，防止惊群，另一个是负载均衡，即不让一个进程处理过多的连接操作。
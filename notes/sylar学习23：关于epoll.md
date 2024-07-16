# sylar学习23：关于epoll

这里重点且全面的学习epoll。因为感觉epoll的逻辑sylar多处可以用到。所以学习启发一下。

我们知道，关于epoll，主要就是三个函数：epoll_create、epoll_ctl、epoll_wait。

以及两个主要用到的数据结构：

```c++
struct eventpoll {
	/* Protect the access to this structure */
	spinlock_t lock; //自旋锁,提供更细粒度的锁定,同时保护对这个结构的访问,比如向rdllist中添加数据等

	/*
	 * This mutex is used to ensure that files are not removed
	 * while epoll is using them. This is held during the event
	 * collection loop, the file cleanup path, the epoll file exit
	 * code and the ctl operations.
	 */
	struct mutex mtx; 
	//最重要的作用就是确保文件描述符不会在使用时被删除,
	//在遍历rdllist,ctl操作中都会使用,这也告诉我们epoll是线程安全的

	/* Wait queue used by sys_epoll_wait() */
	wait_queue_head_t wq; //在epoll_wait中使用的等待队列,把进行epoll_wait的进程都加进去

	/* Wait queue used by file->poll() */
	wait_queue_head_t poll_wait;  //用于epollfd本身被poll的时候

	/* List of ready file descriptors */
	struct list_head rdllist; //就绪的文件描述符链表

	/* RB tree root used to store monitored fd structs */
	struct rb_root rbr; //存储被监听的fd结构的红黑树根

	/*
	 * This is a single linked list that chains all the "struct epitem" that
	 * happened while transferring ready events to userspace w/out
	 * holding ->lock.
	 */
	struct epitem *ovflist; 
	//我们在epoll_wait被唤醒后要把rdllist中的数据发往用户空间,
	//但可能这是也来了被触发的fd结构,这个结构的作用就是在使用rdllist期间把到来的fd结构加到ovflist中,这样可以不必加锁.

	/* The user that created the eventpoll descriptor */
	struct user_struct *user;
};
```



```c++
/*
 * Each file descriptor added to the eventpoll interface will
 * have an entry of this type linked to the "rbr" RB tree.
 */
 //Epoll每次被加入一个fd的时候就会创建一个epitem结构,表示一个被监听的fd.
struct epitem {
	/* RB tree node used to link this structure to the eventpoll RB tree */
	struct rb_node rbn; 
	//对应的红黑树节点,在epoll中已红黑树为主要结构管理fd,而每个fd对应一个epitem,其root保存在eventpoll,上面提到了,即rbr

	/* List header used to link this structure to the eventpoll ready list */
	struct list_head rdllink; //事件的就绪队列,已就绪的epitem会连接在rdllist中,

	/*
	 * Works together "struct eventpoll"->ovflist in keeping the
	 * single linked chain of items.
	 */
	struct epitem *next;

	/* The file descriptor information this item refers to */
	struct epoll_filefd ffd; //此epitem对应的fd和文件指针,用在红黑树中的比较操作,就相当于正常排序的小于号

	/* Number of active wait queue attached to poll operations */
	int nwait; //记录了poll触发回调的次数 epoll_ctl中有提及
	/* List containing poll wait queues */
	struct list_head pwqlist; //保存着被监视fd的等待队列

	/* The "container" of this item */
	struct eventpoll *ep;//该项属于哪个主结构体（多个epitm从属于一个eventpoll)

	/* List header used to link this item to the "struct file" items list */
	//这个实在没搞懂什么意思,参考别的博主的,
	struct list_head fllink; //file中有f_ep_link,用作连接所有监听file的epitem,链表加入的成员为fllink

	/* The structure that describe the interested events and the source fd */
	struct epoll_event event; //所注册的事件,和我们用户空间中见到的差不多
```

首先来看**epoll_create**函数。

```C++
SYSCALL_DEFINE1(epoll_create, int, size)
{
//我们可以看到epoll_create会在内部调用epoll_create1,参数没有什么用,
//所以我们编写代码的时候完全可以直接使用epoll_create1,还省一次函数调用
	if (size <= 0)
		return -EINVAL;

	return sys_epoll_create1(0);
}
SYSCALL_DEFINE1(epoll_create1, int, flags)
{
	int error;
	struct eventpoll *ep = NULL;

	/* Check the EPOLL_* constant for consistency.  */
	BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);

	if (flags & ~EPOLL_CLOEXEC)
		return -EINVAL;
	/*
	 * Create the internal data structure ("struct eventpoll").
	 */
	error = ep_alloc(&ep); //为一个eventpoll指针分配空间,并对成员初始化,下面会细说
	if (error < 0)
		return error;
	/*
	 * Creates all the items needed to setup an eventpoll file. That is,
	 * a file structure and a free file descriptor.
	 */
	error = anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC)); //这里会创建一个匿名文件,并返回文件描述符,下面会细讲
				 //还有一点其实值得一提,就是我们在创建epoll时经常会设置flag位为O_CLOEXEC,这样看来是不必要的 因为内核中已经帮我们做了这件事了
	if (error < 0)
		ep_free(ep);

	return error;
}
```

首先库函数就是这样表示函数的：`SYSCALL_DEFINE1(epoll_create, int, size)`。这个库函数是直接调用epoll_create1的，而且并没有用到参数size。

可以看到其实主体为`ep_alloc(&ep)`，即初始化ep，一个eventpoll类型的变量。

对于函数`ep_alloc`：

```c++
//这个函数其实就是在申请空间后进行一系列的初始化
static int ep_alloc(struct eventpoll **pep) //二级指针,改变指针的值,使用指针的指针
{
	int error;
	struct user_struct *user;
	struct eventpoll *ep;

	user = get_current_user();
	error = -ENOMEM;
	ep = kzalloc(sizeof(*ep), GFP_KERNEL);//申请空间,且在申请后清零
	if (unlikely(!ep))
		goto free_uid;

	spin_lock_init(&ep->lock);
	mutex_init(&ep->mtx);
	init_waitqueue_head(&ep->wq);
	init_waitqueue_head(&ep->poll_wait);
	INIT_LIST_HEAD(&ep->rdllist);
	ep->rbr = RB_ROOT;
	ep->ovflist = EP_UNACTIVE_PTR; //初始化为EP_UNACTIVE_PTR,epoll_wait中会提到其用处,用于避免惊群
	ep->user = user;

	*pep = ep;

	return 0;

free_uid:
	free_uid(user);
	return error;
}

void init_waitqueue_head(wwait_queue_head_t *q)
{
    spin_lock_init(&q->lock);
    INIT_LIST_HEAD(&q->task_list);
}
```

其实这里就是对`eventpoll *ep`的初始化。

下一个部分为函数`error = anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep, O_RDWR | (flags & O_CLOEXEC));`

```c++
int anon_inode_getfd(const char *name, const struct file_operations *fops,
		     void *priv, int flags)
{
	int error, fd;
	struct file *file;

	error = get_unused_fd_flags(flags);
	if (error < 0)
		return error;
	fd = error;

	//这里做了一个优化,即当一个文件不需要完整的索引节点即可正常运行的时候,就使得它们链接到同一个inode上
	//创建一个匿名文件,所有由anon_inode_getfile创建的文件都指向一个inode
	file = anon_inode_getfile(name, fops, priv, flags);  //其中把private_data 设置为priv
	if (IS_ERR(file)) {
		error = PTR_ERR(file);
		goto err_put_unused_fd;
	}
	fd_install(fd, file); //把一个fd与一个文件指针相连接

	return fd;

err_put_unused_fd:
	put_unused_fd(fd);
	return error;
}
```

这里是创建一个匿名文件，与我们创建的`eventpoll *ep`进行关联，返回文件描述符，因为我们最后是要返回一个句柄的嘛。

`get_unused_fd_flags`为一个宏：

```C++
#define get_used_fd_flags(flags) alloc_fd(0, (flags))
```

关于函数alloc_fd：

```C++
//这个函数其实就是在当前进程内创建了一个文件描述符并返回
int alloc_fd(unsigned start, unsigned flags)
{	//我们可以看到这是从当前进程获取文件描述符
	struct files_struct *files = current->files;
	unsigned int fd;
	int error;
	struct fdtable *fdt;

	spin_lock(&files->file_lock); 
repeat:
	fdt = files_fdtable(files); //打开进程的文件描述符表
	fd = start;
	if (fd < files->next_fd)
		fd = files->next_fd;

	if (fd < fdt->max_fds)
		//寻找一个空闲的bit位,也就是寻找一个未使用的文件描述符
		fd = find_next_zero_bit(fdt->open_fds->fds_bits,
					   fdt->max_fds, fd);
	
	//通过fd值判断是否需要扩展文件描述符表
	error = expand_files(files, fd);
	if (error < 0)
		goto out;
	//下面就是一些排错机制了
	/*
	 * If we needed to expand the fs array we
	 * might have blocked - try again.
	 */
	if (error)
		goto repeat;

	if (start <= files->next_fd)
		files->next_fd = fd + 1;

	FD_SET(fd, fdt->open_fds);
	if (flags & O_CLOEXEC)
		FD_SET(fd, fdt->close_on_exec);
	else
		FD_CLR(fd, fdt->close_on_exec);
	error = fd;
#if 1
	/* Sanity check */
	if (rcu_dereference_raw(fdt->fd[fd]) != NULL) {
		printk(KERN_WARNING "alloc_fd: slot %d not NULL!\n", fd);
		rcu_assign_pointer(fdt->fd[fd], NULL);
	}
#endif

out:
	spin_unlock(&files->file_lock);
	return error;
}
```

这个函数其实就是在当前进程内创建了一个文件描述符并返回。

`anon_inode_getfile`相当于进行创建文件，把`private_data` 设置为`eventpoll *ep`，之后再通过`fd_install(fd, file)`把上面生成的文件描述符和这个匿名文件相关联。

总结一下epoll_create函数：

> 1.创建eventpoll *ep，并进行初始化。
>
> 2.在当前文件创建一个文件描述符fd。
>
> 3.创建文件，将private_data设置为eventpoll *ep。
>
> 4.将创建的fd与创建的文件相关联。

需要注意的是，在函数`anon_inode_getfile`中进行了一个小小的优化，即所有的epoll虽然fd不同，但file指针共享一个inode，即索引节点。



之后是**epoll_ctl**函数，进行注册。

```c++
/*
 * epfd 为epoll_create创建的fd
 * op 是我们的操作种类 增删改 ADD DEL MOD
 * fd 当然是我们要加入的fd了
 * event 我们所关心的事件类型 注意只有我们注册的事件才会在epoll_wait被唤醒后传递到用户空间 否则虽然内核可以收到 但不会传递到用户空间
 */
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
	int error;
	int did_lock_epmutex = 0;
	struct file *file, *tfile;
	struct eventpoll *ep;
	struct epitem *epi;
	struct epoll_event epds;

	error = -EFAULT;
	/*
	static inline int ep_op_has_event(int op)
	{
		return op != EPOLL_CTL_DEL; // DEL的时候当然不用从用户态拷贝event啦
	}
*/
	if (ep_op_has_event(op) && //我们可以在上面看到ep_op_has_event的函数体 用以判断是否需要从用户态拷贝数据
	    copy_from_user(&epds, event, sizeof(struct epoll_event))) //从用户空间把数据拷贝过来
		goto error_return;

	/* Get the "struct file *" for the eventpoll file */
	error = -EBADF;
	file = fget(epfd); //得到epoll的file指针
	if (!file)
		goto error_return;

	/* Get the "struct file *" for the target file */
	tfile = fget(fd); //得到要添加fd的file指针
	if (!tfile)
		goto error_fput;

	/* The target file descriptor must support poll */
	error = -EPERM;
	if (!tfile->f_op || !tfile->f_op->poll) //如果要监听的fd不支持poll的话就没办法了 只能报error了.
		goto error_tgt_fput;

	/*
	 * We have to check that the file structure underneath the file descriptor
	 * the user passed to us _is_ an eventpoll file. And also we do not permit
	 * adding an epoll file descriptor inside itself.
	 */
	error = -EINVAL;
	/*
/*
static inline int is_file_epoll(struct file *f)
{
	return f->f_op == &eventpoll_fops; //更快的检测fd是否为一个epoll_fd
}
*/
	if (file == tfile || !is_file_epoll(file)) //当然不允许自己监听自己了,同时得保证epfd是一个epoll的fd
		goto error_tgt_fput;

	/*
	 * At this point it is safe to assume that the "private_data" contains
	 * our own data structure.
	 */
	ep = file->private_data; //上一篇说到private_data中存的是分配好的eventpoll

	/*
	 * When we insert an epoll file descriptor, inside another epoll file
	 * descriptor, there is the change of creating closed loops, which are
	 * better be handled here, than in more critical paths.
	 *
	 * We hold epmutex across the loop check and the insert in this case, in
	 * order to prevent two separate inserts from racing and each doing the
	 * insert "at the same time" such that ep_loop_check passes on both
	 * before either one does the insert, thereby creating a cycle.
	 */
	if (unlikely(is_file_epoll(tfile) && op == EPOLL_CTL_ADD)) { //防止epoll插入到另一个epoll中
		mutex_lock(&epmutex);
		did_lock_epmutex = 1;
		error = -ELOOP;
		if (ep_loop_check(ep, tfile) != 0)
			goto error_tgt_fput;
	}


	mutex_lock(&ep->mtx); //对fd进行操作时上mtx这把锁

	/*
	 * Try to lookup the file inside our RB tree, Since we grabbed "mtx"
	 * above, we can be sure to be able to use the item looked up by
	 * ep_find() till we release the mutex.
	 */
	epi = ep_find(ep, tfile, fd); //O(logn)查询要操作的fd是否已经存在
	/*
		static inline void ep_set_ffd(struct epoll_filefd *ffd,
					      struct file *file, int fd) //创建一个进行比较的项
		{
			ffd->file = file;
			ffd->fd = fd;
		}
		
		
		static inline int ep_cmp_ffd(struct epoll_filefd *p1,
					     struct epoll_filefd *p2)
		{
			return (p1->file > p2->file ? +1:
			        (p1->file < p2->file ? -1 : p1->fd - p2->fd));
		}
		//从RB-tree中寻找期望插入的fd是否存在
		static struct epitem *ep_find(struct eventpoll *ep, struct file *file, int fd)
		{ 
			int kcmp;
			struct rb_node *rbp;
			struct epitem *epi, *epir = NULL;
			struct epoll_filefd ffd;
		
			ep_set_ffd(&ffd, file, fd);
			for (rbp = ep->rbr.rb_node; rbp; ) {
				epi = rb_entry(rbp, struct epitem, rbn);
				kcmp = ep_cmp_ffd(&ffd, &epi->ffd); //红黑树的排序规则
				if (kcmp > 0)
					rbp = rbp->rb_right;
				else if (kcmp < 0)
					rbp = rbp->rb_left;
				else {
					epir = epi;
					break;
				}
			}
			return epir; //如果找到返回找到的fd所对应的节点
		}
	*/

	error = -EINVAL;
	switch (op) {
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds.events |= POLLERR | POLLHUP; //我们可以看到这两个事件显然我们也没有必要注册
			error = ep_insert(ep, &epds, tfile, fd);
		} else
			error = -EEXIST;
		break;
	case EPOLL_CTL_DEL:
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD:
		if (epi) {
			epds.events |= POLLERR | POLLHUP;
			error = ep_modify(ep, epi, &epds);
		} else
			error = -ENOENT;
		break;
	}
	mutex_unlock(&ep->mtx);

error_tgt_fput:
	if (unlikely(did_lock_epmutex))
		mutex_unlock(&epmutex);

	fput(tfile); //正常关闭文件
error_fput:
	fput(file);
error_return:

	return error;
}
```

现在来看这个函数好简单hh。

一开始都是各种准备，准备要进行操作的元素。

```c++
file = fget(epfd); //得到epoll的file指针
tfile = fget(fd); //得到要添加fd的file指针
ep = file->private_data; //上一篇说到private_data中存的是分配好的eventpoll
```

如果上面准备这些，以及进行了很多判断/

之后主要的就是检查这次要操作的fd是否已经存在。我们知道eventpoll是用红黑树来储存fd的，所以这里查找就是按照搜索树的方式进行的。

```c++
if (kcmp > 0)
	rbp = rbp->rb_right;
else if (kcmp < 0)
	rbp = rbp->rb_left;
else {
		epir = epi;
		break;
	}
```

这里看的比较清晰。

然后之后就是根据传入的操作参数op进行判断要进行什么样的操作，这里有三种可能，分别是ADD DEL MOD。

就像我们遇到的ADD，就是要进行函数ep_insert

```c++
//注意 这个整个函数调用都是上锁的
static int ep_insert(struct eventpoll *ep, struct epoll_event *event,
		     struct file *tfile, int fd)
{
	int error, revents, pwake = 0;
	unsigned long flags;
	long user_watches;
	struct epitem *epi;
	struct ep_pqueue epq;
	
	//是否到达最大的监听数
	user_watches = atomic_long_read(&ep->user->epoll_watches);
	if (unlikely(user_watches >= max_user_watches))
		return -ENOSPC;
	if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL))) //最有意思的地方 slab机制,在缓存中分配对象,为了更快的访问
		return -ENOMEM;

	/* Item initialization follow here ... */ //正如内核注释所言 正常的初始化
	INIT_LIST_HEAD(&epi->rdllink);
	INIT_LIST_HEAD(&epi->fllink);
	INIT_LIST_HEAD(&epi->pwqlist);
	epi->ep = ep;
	ep_set_ffd(&epi->ffd, tfile, fd);
	epi->event = *event;
	epi->nwait = 0;
	epi->next = EP_UNACTIVE_PTR;

	/* Initialize the poll table using the queue callback */
	epq.epi = epi;
	//设置poll的回调为ep_ptable_queue_proc,这其中藏着epoll如此高效的原因!
	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

	/*
	 * Attach the item to the poll hooks and get current event bits.
	 * We can safely use the file* here because its usage count has
	 * been increased by the caller of this function. Note that after
	 * this operation completes, the poll callback can start hitting
	 * the new item.
	 */
	//进行完这一步 算是把epitem和这个fd连接起来了 会在事件到来的时候执行上面注册的回调
	revents = tfile->f_op->poll(tfile, &epq.pt);

	/*
	 * We have to check if something went wrong during the poll wait queue
	 * install process. Namely an allocation for a wait queue failed due
	 * high memory pressure.
	 */
	error = -ENOMEM;
	if (epi->nwait < 0)
		goto error_unregister;

	/* Add the current item to the list of active epoll hook for this file */
	spin_lock(&tfile->f_lock); //tfile是要加入的fd的file指针
	list_add_tail(&epi->fllink, &tfile->f_ep_links); //file会把监听自己的epitem连接起来
	spin_unlock(&tfile->f_lock);

	/*
	 * Add the current item to the RB tree. All RB tree operations are
	 * protected by "mtx", and ep_insert() is called with "mtx" held.
	 */
	ep_rbtree_insert(ep, epi); //向ep的红黑树中插入一个epitem

	/* We have to drop the new item inside our item list to keep track of it */
	spin_lock_irqsave(&ep->lock, flags);

	/* If the file is already "ready" we drop it inside the ready list */
	//这里处理的是监听的fd刚好有事件发生,如果现在不处理,可能后面不会到来数据,也就刽触发回调,它们可能就再也不会被处理了
	if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
		list_add_tail(&epi->rdllink, &ep->rdllist);

		/* Notify waiting tasks that events are available */
		if (waitqueue_active(&ep->wq))
			wake_up_locked(&ep->wq); //唤醒epoll_wait中的等待队列
		if (waitqueue_active(&ep->poll_wait))
			pwake++;
	}

	spin_unlock_irqrestore(&ep->lock, flags);

	atomic_long_inc(&ep->user->epoll_watches);

	/* We have to call this outside the lock */
	if (pwake)
		ep_poll_safewake(&ep->poll_wait);

	return 0;

error_unregister:
	ep_unregister_pollwait(ep, epi);

	/*
	 * We need to do this because an event could have been arrived on some
	 * allocated wait queue. Note that we don't care about the ep->ovflist
	 * list, since that is used/cleaned only inside a section bound by "mtx".
	 * And ep_insert() is called with "mtx" held.
	 */
	spin_lock_irqsave(&ep->lock, flags);
	if (ep_is_linked(&epi->rdllink))
		list_del_init(&epi->rdllink);
	spin_unlock_irqrestore(&ep->lock, flags);

	kmem_cache_free(epi_cache, epi);

	return error;
}
```

在插入函数，首先构造epitem结构，各种初始化，还使用了slab机制在缓存中分配对象，为了更快的访问。

这里还构造了ep_pqueue结构epq。

```c++
/* 轮询队列使用的包装器结构 */
struct ep_pqueue {
	poll_table pt;
	struct epitem *epi;
};
```

```c++
init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
```

相当于执行了`*poll_table->_qproc = ep_ptable_queue_proc*`，即设置poll的回调为ep_ptable_queue_proc，在执行poll_wait时会执行回调。

```c++
revents = tfile->f_op->poll(tfile, &epq.pt);
```

然后对tfile的等待队列调用poll_wait(最后总会执行，然后执行回调函数)，将epitem和这个fd连接起来了。

之后就将这个epitem加入到红黑树中。

如果监听的fd已经来了事件，那么就立刻进行处理：

```c++
if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
    /* 将当前的epitem加入到ready list中去 */
    list_add_tail(&epi->rdllink, &ep->rdllist);
    /* Notify waiting tasks that events are available */
    /* 谁在epoll_wait, 就唤醒它... */
    if (waitqueue_active(&ep->wq))
        wake_up_locked(&ep->wq);
    /* 谁在epoll当前的epollfd, 也唤醒它... */
    if (waitqueue_active(&ep->poll_wait))
        pwake++;
}
```

上面的回调函数`ep_ptable_queue_proc`将epitem和指定的fd使用等待队列关联起来。

```c++
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
				 poll_table *pt)
{
	struct epitem *epi = ep_item_from_epqueue(pt);
	struct eppoll_entry *pwq;
/*

struct eppoll_entry { //这个结构体就是做一个epitem和其回调之间的关联
	// List header used to link this structure to the "struct epitem" 
	struct list_head llink;

	// The "base" pointer is set to the container "struct epitem" 
	struct epitem *base;

	 //Wait queue item that will be linked to the target file wait
	 //queue head.
	wait_queue_t wait;

	// The wait queue head that linked the "wait" wait queue item
	wait_queue_head_t *whead;
};

*/
	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
		//初始化一个等待队列的节点 其中注册的回调为ep_poll_callback
		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
		pwq->whead = whead;  //这个就是监控的fd的等待队列头
		pwq->base = epi; /* 保存epitem */
		add_wait_queue(whead, &pwq->wait);//把初始化的等待对列头插入到所监控的fd的等待对列中,稍后调用
		list_add_tail(&pwq->llink, &epi->pwqlist);
		epi->nwait++;//记录了poll执行回调的次数,即数据来了几次
	} else {
		/* We have to signal that an error occurred */
		epi->nwait = -1;
	}
}
```

当我们检测的fd发生状态改变时，`ep_poll_callback`会被调用。

```C++
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key) //key是events
{
	int pwake = 0;
	unsigned long flags;
	struct epitem *epi = ep_item_from_wait(wait); //由等待队列中获取对应的epitem
	struct eventpoll *ep = epi->ep; //从而得到epoll的结构

	spin_lock_irqsave(&ep->lock, flags); //这里要操作rdllist 上锁

	/*
	 * If the event mask does not contain any poll(2) event, we consider the
	 * descriptor to be disabled. This condition is likely the effect of the
	 * EPOLLONESHOT bit that disables the descriptor when an event is received,
	 * until the next EPOLL_CTL_MOD will be issued.
	 */
	 //#define EP_PRIVATE_BITS (EPOLLONESHOT | EPOLLET)
	if (!(epi->event.events & ~EP_PRIVATE_BITS)) //我们看到如果所监控的fd没有注册什么事件的话,就不加入rdllist中
		goto out_unlock;

	/*
	 * Check the events coming with the callback. At this stage, not
	 * every device reports the events in the "key" parameter of the
	 * callback. We need to be able to handle both cases here, hence the
	 * test for "key" != NULL before the event match test.
	 */
	if (key && !((unsigned long) key & epi->event.events)) //所触发的事件我们没有注册的话当然也不加入rdllist中
		goto out_unlock;

	/*
	 * If we are trasfering events to userspace, we can hold no locks
	 * (because we're accessing user memory, and because of linux f_op->poll()
	 * semantics). All the events that happens during that period of time are
	 * chained in ep->ovflist and requeued later on.
	 */
    //另一种解释
    /*
 * 如果该callback被调用的同时, epoll_wait()已经返回了，也就是说, 此刻应用程序有可能已经在循环获取events
 * 这种情况下, 内核将此刻发生event的epitem用一个单独的链表链起来, 不发给应用程序, 也不丢弃
 * 而是在下一次epoll_wait时返回给用户。
 * 
 * ovflist初始化为EP_UNACTIVE_PTR，在epoll_wait()中，会将其置为NULL
 * epi->next初始化也为EP_UNACTIVE_PTR，在此时会置为空；再处理ovflist时epi->next会置为EP_UNACTIVE_PTR
*/
	 /*
	  * 这里就比较有意思了
	  * 在epoll_wait中EP_UNACTIVE_PTR初始化为EP_UNACTIVE_PTR,而当epoll_wait]中被唤醒后处理rdlist时会将ep->ovflist置为NULL
	  * 也就是说如果ep->ovflist != EP_UNACTIVE_PTR意味这epoll_wait已被唤醒 正在执行loop
	  * 此时我们就把在rdllist遍历时发生的事件用ovflist串起来,在遍历结束后插入rdllist中
	  */
	if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) {
		if (epi->next == EP_UNACTIVE_PTR) {
			epi->next = ep->ovflist;
			ep->ovflist = epi;
		}
		goto out_unlock;
	}

	/* If this file is already in the ready list we exit soon */
	//将当前的epitem加入到rdllist中
	//这就是epoll如此高效的原因,我们不必每次去遍历fd寻找触发的事件,触发事件时会触发回调自动把epitem加入到rdllist中,
	//这使得复杂度从O(N)降到了O(有效事件集合),且我们不必每次注册事件,仅在epoll_ctl(ADD)中注册一次即可(修改除外),
	if (!ep_is_linked(&epi->rdllink)) 
		list_add_tail(&epi->rdllink, &ep->rdllist);

	/*
	 * Wake up ( if active ) both the eventpoll wait list and the ->poll()
	 * wait list.
	 */
	 //唤醒epoll_wait
	if (waitqueue_active(&ep->wq))
		wake_up_locked(&ep->wq);
	if (waitqueue_active(&ep->poll_wait))
		pwake++;

out_unlock:
	spin_unlock_irqrestore(&ep->lock, flags);

	/* We have to call this outside the lock */
	if (pwake)
		ep_poll_safewake(&ep->poll_wait);

	return 1;
}
```

所以其实就是回调函数，即我们检测到fd状态发生变化时，就使用回调。

>epoll_ctl我们调了ep_insert来讲,这是因为其基本道出了epoll如此高效的原因。
>
>首先是epitem的创建使用了slab机制,利用缓存来加速我们的操作。
>
>其次是红黑树,这使得我们的增删改操作的复杂度均为O(logn)。
>
>最重要的一点就是回调的设计,epoll会在插入一个节点的时候创建一个epoll_entry,然后在其中注册ep_ptable_queue_proc,fd会在poll的时候执行这个回调,同时调用ep_poll_callback,这使得被监控的fd被触发时直接就把epitem加入到rdllist中,这样rdllist中就全部都是就绪的fd了! 在有大量不活跃fd时有着显著的性能提升,除此之外我们还不必每次注册事件,这些都是在epitem中保存好的.



最后时**epoll_wait**函数。

```c++
/*
 * Implement the event wait interface for the eventpoll file. It is the kernel
 * part of the user space epoll_wait(2).
 */
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
		int, maxevents, int, timeout)
{
	int error;
	struct file *file;
	struct eventpoll *ep; 
	
	//这个函数中基本是对用户传进来的参数进行一些正确性检验,因为内核对于用户态是不信任的,这也就是干什么都要拷贝的原因吧.

	/* The maximum number of event must be greater than zero */
	if (maxevents <= 0 || maxevents > EP_MAX_EVENTS) //判断最大事件数是否在正确范围内
		return -EINVAL;

	/* Verify that the area passed by the user is writeable */
	//检测传用户传入的指针所指区域是否可写
	if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event))) {
		error = -EFAULT;
		goto error_return;
	}

	/* Get the "struct file *" for the eventpoll file */
	error = -EBADF;
	file = fget(epfd); //获取epoll的file结构体 linux的文件概念真滴是强大
	if (!file)
		goto error_return;

	/*
	 * We have to check that the file structure underneath the fd
	 * the user passed to us _is_ an eventpoll file.
	 */
	error = -EINVAL;
	if (!is_file_epoll(file)) //判断这是不是一个epoll的文件指针
		goto error_fput;

	/*
	 * At this point it is safe to assume that the "private_data" contains
	 * our own data structure.
	 */
	ep = file->private_data; //从file的private_data中获取eventpoll结构

	/* Time to fish for events ... */
	error = ep_poll(ep, events, maxevents, timeout); //epoll_wait的主函数体

error_fput:
	fput(file);
error_return:

	return error;
}
```

嗷其实主体是ep_poll函数，其他都是些检查。

```c++
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, int maxevents, long timeout)
{
	int res, eavail;
	unsigned long flags;
	long jtimeout;
	wait_queue_t wait;

	/*
	 * Calculate the timeout by checking for the "infinite" value ( -1 )
	 * and the overflow condition. The passed timeout is in milliseconds,
	 * that why (t * HZ) / 1000.
	 */
	 //计算睡眠事件 要转换成毫秒 式子里那个+999)/1000的作用是为了向上取整 HZ在/drivers/md/raid6.h中定义 值为1000
	 //所以这里面的向上取整是否真的有必要呢?
	jtimeout = (timeout < 0 || timeout >= EP_MAX_MSTIMEO) ?
		MAX_SCHEDULE_TIMEOUT : (timeout * HZ + 999) / 1000;

retry: //这个标记很重要 下面会说道

	//#define write_lock_irqsave(lock, flags)	flags = _write_lock_irqsave(lock)
	//#define _write_lock_irqsave(lock, flags)	__LOCK_IRQSAVE(lock, flags)
	/*
	#define __LOCK_IRQSAVE(lock, flags) \
  	do { local_irq_save(flags); __LOCK(lock); } while (0)
	*/
	write_lock_irqsave(&ep->lock, flags); //自旋锁上锁 一系列宏上面已列出

	res = 0;
	if (list_empty(&ep->rdllist)) { //如果就绪链表为空的话就进入if 也就是睡眠了,否则的话直接跳出,
		//相当于我们如果在epoll_ctl(ADD)后,事件已经发生了后在wait,消耗实际上就只是一个用户态到内核态的转换和拷贝而已,
		//不涉及从等待队列中唤醒
		/*
		 * We don't have any available event to return to the caller.
		 * We need to sleep here, and we will be wake up by
		 * ep_poll_callback() when events will become available.
		 */
		init_waitqueue_entry(&wait, current); //用当前进程初始化一个等待队列的entry
		add_wait_queue(&ep->wq, &wait);
		//把刚刚初始化的这个等待队列节点加到epoll内部的等待队列中去,也就是说在epoll_wait被唤醒时唤醒本进程

		for (;;) { //开始睡眠
			/*
			 * We don't want to sleep if the ep_poll_callback() sends us
			 * a wakeup in between. That's why we set the task state
			 * to TASK_INTERRUPTIBLE before doing the checks.
			 */
			 //这里我翻译下源码中的解释,
			 //我们不希望ep_poll_callback()发送给我们wakeup消息时我们还在沉睡,这就是为什么我们我们要在检查前设置成TASK_INTERRUPTIBLE
			set_current_state(TASK_INTERRUPTIBLE);
			if (!list_empty(&ep->rdllist) || !jtimeout) //rdllist不为空或者超时
				break;
			//当阻塞于某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个EINTR错误
			if (signal_pending(current)) { //收到一个信号的时候也可能被唤醒
				res = -EINTR;
				break;
			}

			write_unlock_irqrestore(&ep->lock, flags); //解锁 开始睡眠
			jtimeout = schedule_timeout(jtimeout);//schedule_timeout的功能源码中的介绍为 sleep until timeout
			write_lock_irqsave(&ep->lock, flags);
		}
		remove_wait_queue(&ep->wq, &wait); //醒来啦!从等待队列中移除

		set_current_state(TASK_RUNNING);
	}
	/* Is it worth to try to dig for events ? */

	//这里要注意 当rdlist被其他进程访问的时候,ep->ovflist会被设置为NULL,那时rdlist会被txlist替换,
	//因为在遍历rdlist的时候有可能ovlist传入数据,然后写入rdllist,所以rdllist也有可能不为空,
	//但为空且其他进程在访问的时候就会将eavail设置为true,为后面goto到retry再次进行睡眠做准备,
	//这样就避免了用户态的唤醒,从而避免了一定程度上的惊群.文末附上我对惊群的测试于结论的链接
    eavail = !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;

	write_unlock_irqrestore(&ep->lock, flags);

	/*
	 * Try to transfer events to user space. In case we get 0 events and
	 * there's still timeout left over, we go trying again in search of
	 * more luck.
	 */
	if (!res && eavail &&
	    !(res = ep_send_events(ep, events, maxevents)) && jtimeout)
	   //res还得为0 epoll才会继续沉睡,有可能ovflist其中有数据 ,后面赋给了rdllist,这是rdllist有数据,也就是说此时epoll_wait醒来还有数据,所以不必继续沉睡
		goto retry;

	return res;
}
```


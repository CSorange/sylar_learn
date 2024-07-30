# sylar学习23：关于epoll

这里重点且全面的学习epoll。因为感觉epoll的逻辑sylar多处可以用到。所以学习启发一下。

我们知道，关于epoll，主要就是三个函数：epoll_create、epoll_ctl、epoll_wait。

首先先要学一点前导知识：

> 1.等待队列waitqueue
>
> 队列头(wait_queue_head_t)往往是资源生产者,队列成员(wait_queue_t)往往是资源消费者,当头的资源ready后, 会逐个执行每个成员指定的回调函数,来通知它们资源已经ready了。(真的很巧妙www)
>
> 2.内核的poll机制
>
> epoll本身没有给内核引入什么特别复杂的技术，只是已有功能的重新组合，达到了超过select的效果，所以之后还会看到poll，需要好好了解poll机制。
>
> 被Poll的fd, 必须在实现上支持内核的Poll技术,比如fd是某个字符设备,或者是个socket, 它必须实现file_operations(后面会看到)中的poll操作, 给自己分配有一个等待队列头.主动poll fd的某个进程必须分配一个等待队列成员, 添加到fd的等待队列里面去, 并指定资源ready时的回调函数.
>
> 用socket做例子, 它必须有实现一个poll操作, 这个Poll是发起轮询的代码(应该是关心socket的进程发起的)必须主动调用的, 该函数中必须调用poll_wait(),poll_wait会将发起者作为等待队列成员加入到socket的等待队列中去.这样socket发生状态变化时可以通过队列头逐个通知所有关心它的进程.这一点必须很清楚的理解, 否则会想不明白epoll是如何得知fd的状态发生变化的.
>
> 3.epollfd本身也是个fd, 所以它本身也可以被epoll(eventpoll中有数据结构专门针对这一点)

还有一些知识：(自己没有那么明显的感觉出来很重要的)

>1.fd我们知道是文件描述符, 在内核态, 与之对应的是struct file结构,可以看作是内核态的文件描述符.
>
>2.spinlock, 自旋锁, 必须要非常小心使用的锁。
>
>3.引用计数在内核中是非常重要的概念,内核代码里面经常有些release, free释放资源的函数几乎不加任何锁,这是因为这些函数往往是在对象的引用计数变成0时被调用,既然没有进程在使用在这些对象, 自然也不需要加锁.

以及两个主要用到的数据结构：

```c++
/* 每创建一个epollfd, 内核就会分配一个eventpoll与之对应, 可以说是
 * 内核态的epollfd. */
struct eventpoll {
    /* Protect the this structure access */
    spinlock_t lock;
    /*
     * This mutex is used to ensure that files are not removed
     * while epoll is using them. This is held during the event
     * collection loop, the file cleanup path, the epoll file exit
     * code and the ctl operations.
     */
    /* 添加, 修改或者删除监听fd的时候, 以及epoll_wait返回, 向用户空间
     * 传递数据时都会持有这个互斥锁, 所以在用户空间可以放心的在多个线程
     * 中同时执行epoll相关的操作, 内核级已经做了保护. */
    struct mutex mtx;
    /* Wait queue used by sys_epoll_wait() */
    /* 调用epoll_wait()时, 我们就是"睡"在了这个等待队列上... */
    wait_queue_head_t wq;
    /* Wait queue used by file->poll() */
    /* 这个用于epollfd本身被poll的时候... */
    wait_queue_head_t poll_wait;
    /* List of ready file descriptors */
    /* 所有已经ready的epitem都在这个链表里面 */
    struct list_head rdllist;
    /* RB tree root used to store monitored fd structs */
    /* 所有要监听的epitem都在这里 */
    struct rb_root rbr;
    /*
        这是一个单链表链接着所有的struct epitem当event转移到用户空间时
     */
     * This is a single linked list that chains all the "struct epitem" that
     * happened while transfering ready events to userspace w/out
     * holding ->lock.
     */
    struct epitem *ovflist;
    /* The user that created the eventpoll descriptor */
    /* 这里保存了一些用户变量, 比如fd监听数量的最大值等等 */
    struct user_struct *user;
};
```



```c++
/* epitem 表示一个被监听的fd */
struct epitem {
    /* RB tree node used to link this structure to the eventpoll RB tree */
    /* rb_node, 当使用epoll_ctl()将一批fds加入到某个epollfd时, 内核会分配
     * 一批的epitem与fds们对应, 而且它们以rb_tree的形式组织起来, tree的root
     * 保存在epollfd, 也就是struct eventpoll中.
     * 在这里使用rb_tree的原因我认为是提高查找,插入以及删除的速度.
     * rb_tree对以上3个操作都具有O(lgN)的时间复杂度 */
    struct rb_node rbn;
    /* List header used to link this structure to the eventpoll ready list */
    /* 链表节点, 所有已经ready的epitem都会被链到eventpoll的rdllist中 */
    struct list_head rdllink;
    /*
     * Works together "struct eventpoll"->ovflist in keeping the
     * single linked chain of items.
     */
    /* 这个在代码中再解释... */
    struct epitem *next;
    /* The file descriptor information this item refers to */
    /* epitem对应的fd和struct file */
    struct epoll_filefd ffd;
    /* Number of active wait queue attached to poll operations */
    int nwait;
    /* List containing poll wait queues */
    struct list_head pwqlist;
    /* The "container" of this item */
    /* 当前epitem属于哪个eventpoll */
    struct eventpoll *ep;
    /* List header used to link this item to the "struct file" items list */
    struct list_head fllink;
    /* The structure that describe the interested events and the source fd */
    /* 当前的epitem关系哪些events, 这个数据是调用epoll_ctl时从用户态传递过来 */
    struct epoll_event event;
};
struct epoll_filefd {
    struct file *file;
    int fd;
};
/* poll所用到的钩子Wait structure used by the poll hooks */
struct eppoll_entry {
    /* List header used to link this structure to the "struct epitem" */
    struct list_head llink;
    /* The "base" pointer is set to the container "struct epitem" */
    struct epitem *base;
    /*
     * Wait queue item that will be linked to the target file wait
     * queue head.
     */
    wait_queue_t wait;
    /* The wait queue head that linked the "wait" wait queue item */
    wait_queue_head_t *whead;
};
/* Wrapper struct used by poll queueing */
struct ep_pqueue {
    poll_table pt;
    struct epitem *epi;
};
/* Used by the ep_send_events() function as callback private data */
struct ep_send_events_data {
    int maxevents;
    struct epoll_event __user *events;
};
```

所以其实主要有三个吧，**eventpoll**、**epitem**、**epoll_event**。感觉自己第一个和第三个老是搞混。还有这三个的从属关系，一个A对应多个B，一个B对应一个A，一个C属于一个B。



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

可以看到其实主体为`ep_alloc(&ep)`，即初始化ep，一个eventpoll类型的变量，分配完空间之后开始对内部的元素进行赋值。

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
	init_waitqueue_head(&ep->poll_wait);//这里应该就只是初始化头节点吧，不过这个封装了一个锁(观察init_waitqueue_head函数和INIT_LIST_HEAD函数之间的区别可以看到)
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

epollfd本身并不存在一个真正的文件与之对应, 所以内核需要创建一个"虚拟"的文件, 并为之分配真正的struct file结构, 而且有真正的fd.

这里的参数eventpoll_fops为file_operations类型，直观感受是一个很大的公共数据结构。含义是当你对这个文件(这里是虚拟的)进行操作(比如读)时,fops里面的函数指针指向真正的操作实现, 类似C++里面虚函数和子类的概念.epoll只实现了poll和release(就是close)操作, 其它文件系统操作都有VFS全权处理了.ep就是struct epollevent, 它会作为一个私有数据保存在struct file的private指针里面.其实说白了, 就是为了能通过fd找到struct file, 通过struct file能找到eventpoll结构.

(其实更细节的函数不需要掌握的很清晰的，明白含义就可以了)

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
> 4.将创建的fd与创建的匿名文件相关联。

需要注意的是，在函数`anon_inode_getfile`中进行了一个小小的优化，即所有的epoll虽然fd不同，但file指针共享一个inode，即索引节点。



之后是**epoll_ctl**函数，进行注册，即往里面添加fd。

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
	if (!tfile->f_op || !tfile->f_op->poll) //如果要监听的fd不支持poll的话就没办法了 只能报error了，这其实就对应我们上面说的。
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

	//进行完这一步 算是把epitem和这个fd连接起来了 会在事件到来的时候执行上面注册的回调
	revents = tfile->f_op->poll(tfile, &epq.pt);

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
	//这里处理的是监听的fd已经有事件发生,如果现在不处理,可能后面不会到来数据,也就刽触发回调,它们可能就再也不会被处理了
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

初始化一个poll_table，相当于执行了`*poll_table->_qproc = ep_ptable_queue_proc*`，即设置poll的回调为ep_ptable_queue_proc，在执行poll_wait时会执行回调。

```c++
revents = tfile->f_op->poll(tfile, &epq.pt);
```

这一步很关键, 也比较难懂, 完全是内核的poll机制导致的，首先, f_op->poll()一般来说只是个wrapper, 它会调用真正的poll实现,拿UDP的socket来举例, 这里就是这样的调用流程: f_op->poll(), sock_poll(),udp_poll(), datagram_poll(), sock_poll_wait(), 最后调用到我们上面指定的ep_ptable_queue_proc()这个回调函数...(好深的调用路径...)，这其实也可以感同身受，就像之前fiber的回调一样超级长一样。完成这一步之后，我们的epitem就跟这个socket关联起来了, 当它有状态变化时,会通过ep_poll_callback()来通知。所以这里只是相关联起来了。

之后就将这个epitem加入到红黑树中。

如果监听的fd已经来了事件，那么就立刻进行处理：

```c++
if ((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
    /* 将当前的epitem加入到ready list中去 */
    list_add_tail(&epi->rdllink, &ep->rdllist);
    /* Notify waiting tasks that events are available */
    /* 谁在epoll_wait, 就唤醒它... */
    if (waitqueue_active(&ep->wq))//可以看到就是去等待队列里面去找的
        wake_up_locked(&ep->wq);
    /* 谁在epoll当前的epollfd, 也唤醒它... */
    if (waitqueue_active(&ep->poll_wait))
        pwake++;
}
```

上面的回调函数`ep_ptable_queue_proc`将epitem和指定的fd使用等待队列(waitqueue)关联起来。

这里确实很难懂。首先要知道的是关联的是fd和epitem。等待队列的头是fd持有的，中的元素是一个个epitem，所以相当于是监听的fd状态改变，队列头会被唤醒，指定的回调函数会被调用，这个回调函数就是ep_poll_callback。

```c++
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
				 poll_table *pt)
{
	struct epitem *epi = ep_item_from_epqueue(pt);
	struct eppoll_entry *pwq;


	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
		//初始化等待队列, 指定ep_poll_callback为唤醒时的回调函数,当我们监听的fd发生状态改变时, 也就是队列头被唤醒时,指定的回调函数将会被调用.
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

当我们检测的fd发生状态改变时，`ep_poll_callback`会被调用。首先说明的是，这里的wait，其实表示关联fd和epitem中的epitem，所以下面的epi其实表示对应的epitem。

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
	if (waitqueue_active(&ep->poll_wait)) // 如果epollfd也在被poll, 那就唤醒队列里面的所有成员.
		pwake++;

out_unlock:
	spin_unlock_irqrestore(&ep->lock, flags);

	/* We have to call this outside the lock */
	if (pwake)
		ep_poll_safewake(&ep->poll_wait);

	return 1;
}
```

大概可以明白了吧。

所以其实就是回调函数，即我们检测到fd状态发生变化时，就使用回调。

>epoll_ctl我们调了ep_insert来讲,这是因为其基本道出了epoll如此高效的原因。
>
>首先是epitem的创建使用了slab机制,利用缓存来加速我们的操作。
>
>其次是红黑树,这使得我们的增删改操作的复杂度均为O(logn)。
>
>最重要的一点就是回调的设计,epoll会在插入一个节点的时候创建一个epoll_entry,然后在其中注册ep_ptable_queue_proc,fd会在poll的时候执行这个回调,同时调用ep_poll_callback,这使得被监控的fd被触发时直接就把epitem加入到rdllist中,这样rdllist中就全部都是就绪的fd了! 在有大量不活跃fd时有着显著的性能提升,除此之外我们还不必每次注册事件,这些都是在epitem中保存好的.

这里面有两个回调函数，第一个是ep_ptable_queue_proc，后面执行poll_wait的时候会回调这个函数(所以是得执行poll_wait才可以进行回调，fd和epitem才可以关联)，然后第二个是ep_poll_callback，在监听的fd状态发生变化的时候执行。执行之后就知道fd对应的哪个epitem要发生变化。

有个不太明白的地方是，fd和epitem不是一一对应的么，为啥还有搞一个队列mmm。

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
	error = ep_poll(ep, events, maxevents, timeout); //epoll_wait的主函数体，直接睡觉，等待事件到来

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
	wait_queue_t wait;//等待队列

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

	//这里要注意 当rdlist被其他进程访问的时候,ep->ovflist会被设置为NULL,那时rdlist会被txlist替换,(其他地方没有rdlist的出现，不知道这个是什么意思mmm)
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

这个其实和自己看的read_io好像啊，不过似乎应该先看epoll_wait再看read_io的。这里其实主要分为两部分，一个是准备睡觉了，在大if中，另一个是要实际做点什么，目测在`ep_send_events(ep, events, maxevents)`中，不是很确定。

当链表为空时我们才进入大if中，即要开始准备睡眠了。我们先需要做一些准备，然后进入for()循环中开始睡眠。当然在真正开始睡眠(即`jtimeout = schedule_timeout(jtimeout);`)之前，我们还需要进行判断，就像在我们在睡觉之前还要查看一下微信一样，如果rdllist有，即来数据了，就不睡了，如果收到个信号，也不睡了。只有都没有，才睡的。

另外这里时累计睡眠时间，返回值应该是总时间减去了已经睡眠的时间，睡眠时间为0的时候，我们也退出。

醒来之后就从等待队列中移除。

然后就是`ep_send_events(ep, events, maxevents)`。

```c++
//传入eventpoll结构,用户指定的内存和最大事件数 然后执行ep_send_events_proc回调
static int ep_send_events(struct eventpoll *ep,
              struct epoll_event __user *events, int maxevents)
{
    struct ep_send_events_data esed;
    esed.maxevents = maxevents;
    esed.events = events; 
    return ep_scan_ready_list(ep, ep_send_events_proc, &esed);
}
/*ep_scan_ready_list - Scans the ready list in a way that makes possible for
                       the scan code, to call f_op->poll(). Also allows for
                       O(NumReady) performance.
 */
static int ep_scan_ready_list(struct eventpoll *ep,int (*sproc)(struct eventpoll *, struct list_head *, void *), void *priv)//注意上面调用的回调在这个函数中名为sproc
{
	int error, pwake = 0;
	unsigned long flags;
	struct epitem *epi, *nepi;
	LIST_HEAD(txlist); //初始化一个链表 作用是把rellist中的数据换出来

	mutex_lock(&ep->mtx);//操作时加锁 防止ctl中对结构进行修改

	spin_lock_irqsave(&ep->lock, flags); //加锁
	list_splice_init(&ep->rdllist, &txlist);//这里把所有的epitem都转移到了txlist上，rfllist被清空了，是的被清空了
	ep->ovflist = NULL; //这里设置为NULL
	spin_unlock_irqrestore(&ep->lock, flags);

	/*
	 * Now call the callback function.
	 */
	error = (*sproc)(ep, &txlist, priv); //在这个回调函数里面处理每个epitem，sproc 就是 ep_send_events_proc，就是我们实际上对epitem的处理
	//对整个txlist执行回调,也就是对rdllist执行回调 
	//遍历的时候可能所监控的fd也会执行回调,向把fd加入到rellist中,但那个时候可能这里正在遍历,为了不竞争锁,把数据放到ovflist中

	spin_lock_irqsave(&ep->lock, flags); //加锁,把其中数据放入rdllist

	 //上面提到了 当执行sproc回调的时候可能也会有到来的数据,为了避免那时插入rdllist加锁,把数据放到ovlist中.在执行完后加入rdllist中
    //这里处理的是ovflist，这些epitem都是我们在传递数据给用户空间时监听到了事件
	for (nepi = ep->ovflist; (epi = nepi) != NULL;
	     nepi = epi->next, epi->next = EP_UNACTIVE_PTR) {
		//将这些直接放入到readylist中
		if (!ep_is_linked(&epi->rdllink))
			list_add_tail(&epi->rdllink, &ep->rdllist);
	}
	/*
	 * We need to set back ep->ovflist to EP_UNACTIVE_PTR, so that after
	 * releasing the lock, events will be queued in the normal way inside
	 * ep->rdllist.
	 */
	ep->ovflist = EP_UNACTIVE_PTR; //设置为初始化时的样子

	/*
	 * Quickly re-inject items left on "txlist".
	 */
	 //有可能有未处理完的数据,再插入rdllist中,比如说LT
	list_splice(&txlist, &ep->rdllist);

	if (!list_empty(&ep->rdllist)) { //rellist不为空的话,进行唤醒
		/*
		 * Wake up (if active) both the eventpoll wait list and
		 * the ->poll() wait list (delayed after we release the lock).
		 */
		if (waitqueue_active(&ep->wq))
			wake_up_locked(&ep->wq);
		if (waitqueue_active(&ep->poll_wait))
			pwake++;
	}
	spin_unlock_irqrestore(&ep->lock, flags);

	mutex_unlock(&ep->mtx);

	/* We have to call this outside the lock */
	if (pwake)
		ep_poll_safewake(&ep->poll_wait);

	return error;
}
```

关于上面提到的回调函数ep_send_events_proc，这里是直接处理的所有已就绪的epitem。

```c++
/* 该函数作为callbakc在ep_scan_ready_list()中被调用
 * head是一个链表, 包含了已经ready的epitem,
 * 这个不是eventpoll里面的ready list, 而是上面函数中的txlist.
 */
static int ep_send_events_proc(struct eventpoll *ep, struct list_head *head, void *priv)
{
    struct ep_send_events_data *esed = priv;
    int eventcnt;
    unsigned int revents;
    struct epitem *epi;
    struct epoll_event __user *uevent;
    /* 扫描整个链表... */
    for (eventcnt = 0, uevent = esed->events;
         !list_empty(head) && eventcnt < esed->maxevents;) {
        /* 取出第一个成员 */
        epi = list_first_entry(head, struct epitem, rdllink);
        /* 然后从链表里面移除 */
        list_del_init(&epi->rdllink);
        /* 读取events,
         * 注意events我们ep_poll_callback()里面已经取过一次了, 为啥还要再取?
         * 1. 我们当然希望能拿到此刻的最新数据, events是会变的~
         * 2. 不是所有的poll实现, 都通过等待队列传递了events, 有可能某些驱动压根没传
         * 必须主动去读取. */
        revents = epi->ffd.file->f_op->poll(epi->ffd.file, NULL) &
            epi->event.events;
        /*
         * If the event mask intersect the caller-requested one,
         * deliver the event to userspace. Again, ep_scan_ready_list()
         * is holding "mtx", so no operations coming from userspace
         * can change the item.
         */
        if (revents) {
            /* 将当前的事件和用户传入的数据都copy给用户空间,
             * 就是epoll_wait()后应用程序能读到的那一堆数据. */
            if (__put_user(revents, &uevent->events) ||
                __put_user(epi->event.data, &uevent->data)) {
                /* 如果copy过程中发生错误, 会中断链表的扫描,
                 * 并把当前发生错误的epitem重新插入到ready list.
                 * 剩下的没处理的epitem也不会丢弃, 在ep_scan_ready_list()
                 * 中它们也会被重新插入到ready list */
                list_add(&epi->rdllink, head);
                return eventcnt ? eventcnt : -EFAULT;
            }
            eventcnt++;
            uevent++;
            if (epi->event.events & EPOLLONESHOT)
                epi->event.events &= EP_PRIVATE_BITS;
            else if (!(epi->event.events & EPOLLET)) {
/* 嘿嘿, EPOLLET和非ET的区别就在这一步之差呀~
* 如果是ET, epitem是不会再进入到readly list,
* 除非fd再次发生了状态改变, ep_poll_callback被调用.
* 如果是非ET, 不管你还有没有有效的事件或者数据,
* 都会被重新插入到ready list, 再下一次epoll_wait
* 时, 会立即返回, 并通知给用户空间. 当然如果这个
* 被监听的fds确实没事件也没数据了, epoll_wait会返回一个0,
* 空转一次.
*/
                list_add_tail(&epi->rdllink, &ep->rdllist);
            }
        }
    }
    return eventcnt;
}
```

这里其实主要就是`__put_user(revents, &uevent->events)`和`__put_user(epi->event.data, &uevent->data))`，即将当前的事件和用户传入的数据都copy给用户空间。

还有在epoll_create函数中释放资源函数**ep_free**：

```c++
static void ep_free(struct eventpoll *ep)
{
    struct rb_node *rbp;
    struct epitem *epi;
    /* We need to release all tasks waiting for these file */
    if (waitqueue_active(&ep->poll_wait))
        ep_poll_safewake(&ep->poll_wait);
    /*
     * We need to lock this because we could be hit by
     * eventpoll_release_file() while we're freeing the "struct eventpoll".
     * We do not need to hold "ep->mtx" here because the epoll file
     * is on the way to be removed and no one has references to it
     * anymore. The only hit might come from eventpoll_release_file() but
     * holding "epmutex" is sufficent here.
     */
    mutex_lock(&epmutex);
    /*
     * Walks through the whole tree by unregistering poll callbacks.
     */
    for (rbp = rb_first(&ep->rbr); rbp; rbp = rb_next(rbp)) {
        epi = rb_entry(rbp, struct epitem, rbn);
        ep_unregister_pollwait(ep, epi);
    }
    /*
     * Walks through the whole tree by freeing each "struct epitem". At this
     * point we are sure no poll callbacks will be lingering around, and also by
     * holding "epmutex" we can be sure that no file cleanup code will hit
     * us during this operation. So we can avoid the lock on "ep->lock".
     */
    /* 之所以在关闭epollfd之前不需要调用epoll_ctl移除已经添加的fd,
     * 是因为这里已经做了... */
    while ((rbp = rb_first(&ep->rbr)) != NULL) {
        epi = rb_entry(rbp, struct epitem, rbn);
        ep_remove(ep, epi);
    }
    mutex_unlock(&epmutex);
    mutex_destroy(&ep->mtx);
    free_uid(ep->user);
    kfree(ep);
}
```



这里总结一下epoll的优点吧。

> 1.监视的描述符数量不受限制，它所支持的`fd`上限是最大可以打开文件的数目。
>
> 2.IO的效率不会随着监视fd的数量的增长而下降。
>
> 因为epoll不同于select和poll使用的是轮询的方式，而是通过每个fd定义的回调函数来实现的，只有就绪的fd才会执行回调函数ep_poll_callback()。
>
> `ep_poll_callback()`的调用时机是由被监听的`fd`的具体实现, 比如`socket`或者某个设备驱动来决定的,因为等待队列头是他们持有的,`epoll`和当前进程只是单纯的等待。
>
> 3.epoll使用一个文件描述符管理多个描述符。
>
> 将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

而且epoll的数据结构采用的是红黑树，而非select的bitmap，以及poll的数组。
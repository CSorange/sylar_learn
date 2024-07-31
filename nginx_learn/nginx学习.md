## 01.ngx_palloc——内存池

主结构：```ngx_pool_s```

```c++
/**
 * Nginx 内存池数据结构
 */
struct ngx_pool_s {
    ngx_pool_data_t       d; 		/* 内存池的数据区域*/
    size_t                max; 		/* 最大每次可分配内存 */
    ngx_pool_t           *current;  /* 指向当前的内存池指针地址*/
    ngx_chain_t          *chain;	/* 缓冲区链表 */
    ngx_pool_large_t     *large;    /* 存储大数据的链表 */
    ngx_pool_cleanup_t   *cleanup;  /* 可自定义回调函数，清除内存块分配的内存 */
    ngx_log_t            *log;      /* 日志 */
};
```

有一个细节，就是关于s结尾的数据结构与t结尾的数据结构，由于在```ngx_core.h```中使用

```typedef struct ..._s            ..._t;```

所以相当于指向自己的指针。

大结构相当于是一个链表，内存池为一块一块的，使用内存池分配一块区间时，我们使用最快拟合(frist fit)，即如果当前内存池块```current```可以放的下，那就当前放，如果不能，就沿着链表往后找，如果找不到，我们就新建一个内存池块用来放置当前所需的空间。

这里面有两个需要注意的点：

一个是，如果要分配的空间大于一个内存池块的大小，我们则采用第二种分配的内存方式，即让这个空间单独做块，挂载在```large```上面。

关于large：

```c++
//大数据块结构
struct ngx_pool_large_s {
    ngx_pool_large_t     *next;  // 指向下一个存储地址 通过这个地址可以知道当前块长度 
    void                 *alloc; /* 数据块指针地址 */
};
```

另一个是，为了平衡效率与空间，我们会移动```current```的位置。如果要新建一个块时，即表示从```current```到最后一个内存池块都不满足可以放下的条件，即对当前内存块进行```failed++```操作。

当前内存块：

```C++
typedef struct {
    u_char               *last;  /* 内存池中未使用内存的开始节点地址 */
    u_char               *end;   /* 内存池的结束地址 */
    ngx_pool_t           *next;  /* 指向下一个内存池 */
    ngx_uint_t            failed;/* 失败次数 */
} ngx_pool_data_t;
```

如果```failed>4```，即对这个块很失望，那么我们会移动```current```到下一个内存池块。

所以这相当于是一个空间换时间的操作，```current```前面的内存池块进不再进行考虑了，会产生一定的空间浪费，但为了从头开始找，时间太长，因此采用这样的方法。

关于空间的释放，在会话期间是不会回收的，只有会话结束之后，内存池块会整个进行回收。



有一个不太懂的地方是关于回调函数，关于cleanup的处理，自己还不太明白。



```chain```是缓冲区，后面我们会看到缓冲区的构建与释放。



## 02.ngx_array——数组

主结构：ngx_array_t

```c++
typedef struct {
    void        *elts; 		/* 指向数组第一个元素指针*/
    ngx_uint_t   nelts; 	/* 已使用元素的索引*/
    size_t       size; 		/* 每个元素的大小，元素大小固定*/
    ngx_uint_t   nalloc;	/* 分配多少个元素 */
    ngx_pool_t  *pool;  	/* 内存池*/
} ngx_array_t;
```

数组创建的时候直接创造就好，同样是放在内存池的。数组删除也直接删除就好，不过由于我们是不在会话期间释放内存的，所以不会释放占用的资源，不过有一种情况是特例，即这个数组占用的空间正好是此内存池块最后的位置，那么此时就将这部分空间释放(即将内存池未使用位置指针前移覆盖就好)。

由于数据结构与数组实际存放位置可能不在一起(数组存放一定是连续的)，因此我们需要判断两次。

````c++
void
ngx_array_destroy(ngx_array_t *a)
{
    ngx_pool_t  *p;
    p = a->pool;
    if ((u_char *) a->elts + a->size * a->nalloc == p->d.last) {
        p->d.last -= a->size * a->nalloc;
    }
    if ((u_char *) a + sizeof(ngx_array_t) == p->d.last) {
        p->d.last = (u_char *) a;
    }
}

````

在进行添加内容时(可以同一时刻添加一个元素或者多个元素)，如果添加的内容个数没有超过数组容量的上限，那么就直接添加就好。如果达到上限了，分为两种情况：

1. 如果这个数组的末尾正好时所在内存池块未使用空间的开始，且后面的空间时足够的，则直接将未使用指针后移即可。
2. 如果不是，或者未使用的空间不够了，就需要重新开辟空间，同时进行扩容。这里时扩容为原来的两倍(如果n大于数组容量```nalloc```则扩容为2*n)，将数组复制过来，然后再进行添加。同样的，数组还在当前的内存池里。



## 03.ngx_buf——缓冲区

缓存区主要涉及到两个数据结构，分别是```ngx_buf_s```、```ngx_chain_s```。其中```ngx_chain_s```为链式结构，而```ngx_buf_s```为具体内容。可以看到```ngx_buf_s```还存在关于待处理文件的标记，即此结构既可以处理内容，也可以处理文件。

缓冲区可以有多块，然后用链表穿起来，不过这是从属与某一内存池块的，即缓冲区都是连在```pool```的```chain```上的。



## 04.ngx_queue——双向链表

双向链表一大特点就是结构与业务的解耦。其实这样的结构自己之前在中兴实习的时候也见过。

双向链表本身就是一个节点，包括向前的指针与向后的指针。

```C++
struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

当我们想根据业务找到当前元素之前或者之后的元素时，我们就根据数据结构中的偏移量得到```ngx_queue_s```，再沿着双向链表读取即可。

```c++
#define ngx_queue_data(q, type, link)                                         \
    (type *) ((u_char *) q - offsetof(type, link))
```

所以还是要求此数据结构之前的内容占据的数据长度是固定的。

阅读代码可以发现，此双向链表可能是环形的，因为最后一个元素的```next```是第一个元素。



## 05.ngx_list——单向链表

链表的结构很固定，除了链表头的数据结构以外，就是每个链表节点。

```ngx_list_t```链表头：

```C++
typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
```

```ngx_list_part_s```链表节点：

```c++
struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
```

链表节点不需要空间连续。

在添加节点的时候，如果空间不够，就直接再开辟一片空间存放节点，然后接起来就可以了。



## 06.ngx_hash——哈希表

在看哈希表之前需要有两个学习的地方，一个是关于内存对齐。

内存对齐会浪费一定的空间(毕竟我们给占据少数字节的元素开辟了一个较大的空间)，但会节省一定的空间。

另一个关于hash本身。

hash其实有点类似与桶排序，我们会将所有的元素分为若干个桶，然后往桶里面添加元素。当然一个桶里可能不止一个元素(对应关系可能没有那么彻底)，可能会有多个，那么我们就会用链表将这些元素连起来。我们找的时候，到桶里面了，发现不止一个元素，就相当于还要再找一下，于是就会发生哈希碰撞的问题。

nginx中的哈希表其实是静态表，即在初始化的时候就已经把空间以及位置，比如有几个桶，每个桶里面有多少元素都分配好了，桶以及桶里的元素是连续分配的，不过每个桶之后都会跟一个空节点，来标记当前桶已经用完了。

所以初始化的时候我们需要进行动态的确定，即确定以上参数(桶数，桶中元素数)。

对于每一个保存的节点：

```c++
typedef struct {
    void             *value; 	/* 指向value的指针 */
    u_short           len;   	/* key的长度 */
    u_char            name[1]; 	/* 指向key的第一个地址，key长度为变长(设计上的亮点)*/
} ngx_hash_elt_t;
```

每个节点包括value值以及key值。这里的key为变长，我们在搜索时需要先判断长度```len```是否一致，然后再判断每个name值是否一致。

对于hash桶：

```c++
typedef struct {
    ngx_hash_elt_t  **buckets; 	/* hash表的桶指针地址值 */
    ngx_uint_t        size; 	/* hash表的桶的个数*/
} ngx_hash_t;
```

即由桶数组以及桶的个数组成。

这个结构和之前遇到的结构不太一样的地方是：这里需要考虑初始化的，因为构造了一个更上层的数据结构：

```c++
typedef struct {
    ngx_hash_t       *hash;	/* 指向hash数组结构 */
    ngx_hash_key_pt   key;  /* 计算key散列的方法 */
 
    ngx_uint_t        max_size; 	/* 最大多少个 */
    ngx_uint_t        bucket_size; 	/* 桶的存储空间大小 */
 
    char             *name; /* hash表名称 */
    ngx_pool_t       *pool; /* 内存池 */
    ngx_pool_t       *temp_pool; /* 临时内存池*/
} ngx_hash_init_t;
```

中间看到的两个```size```即为我们上面提到的了初始化所需要用到变量。

但这里的动态初始化还没有看太懂mmm



## 07.ngx_string——字符串

nginx中的字符串很简洁：

```c++
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

可以看到就只有长度与字符串内容。

```ngx_keyval_t```为KV结构：

```c++
typedef struct {
    ngx_str_t   key;
    ngx_str_t   value;
} ngx_keyval_t;
```





ngx_show_version = 1;
                ngx_show_help = 1;

ngx_show_version = 1;
                ngx_show_configure = 1;

ngx_test_config = 1;
                ngx_dump_config = 1;

nginx -s reload


原文作者：SHERlocked_93
原文链接：https://www.nginx.org.cn/article/detail/545
转载来源：NGINX开源社区
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

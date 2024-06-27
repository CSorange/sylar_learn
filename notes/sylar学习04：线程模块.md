# sylar学习04：线程模块

该模块主要体现在`thread.h`中。

在文件`thread.h`中只有一个类，即`Thread`。有趣的是，命名thread的时候使用了类`Noncopyable`，观察可以发现，类`Noncopyable`禁用了拷贝构造函数和赋值函数，即不允许这样的操作出现在类thread中。

变量为：

```c++
private:
    /// 线程id
    pid_t m_id = -1;
    /// 线程结构
    pthread_t m_thread = 0;
    /// 线程执行函数
    std::function<void()> m_cb;
    /// 线程名称
    std::string m_name;
    /// 信号量
    Semaphore m_semaphore;
```

在线程中只有信号量的体现，锁模块中其他的互斥量，读写锁，自旋锁，原子锁在`mutex.h`文件中体现。

关于线程id的问题，直接调用`unistd.h`文件中的函数即可。

```c++
//进程pid: getpid()                 
//线程tid: pthread_self()     //进程内唯一，但是在不同进程则不唯一。
//线程pid: syscall(SYS_gettid)     //系统内是唯一的
```

```c++
static thread_local Thread *t_thread          = nullptr;
static thread_local std::string t_thread_name = "UNKNOW";
```

`static thread_local`是C++中的一个关键字组合，用于定义静态线程本地存储变量。具体来说，当一个变量被声明为`static thread_local`时，它会在每个线程中拥有自己独立的静态实例，并且对其他线程不可见。这使得变量可以跨越多个函数调用和代码块，在整个程序运行期间保持其状态和值不变。

需要注意的是，由于静态线程本地存储变量是线程特定的，因此它们的初始化和销毁时机也与普通静态变量不同。具体来说，在每个线程首次访问该变量时会进行初始化，在线程结束时才会进行销毁，而不是在程序启动或运行期间进行一次性初始化或销毁。确实这还是不太一样。

构造函数：

```c++
Thread::Thread(std::function<void()> cb, const std::string &name)
    : m_cb(cb)
    , m_name(name) {
    if (name.empty()) {
        m_name = "UNKNOW";
    }
    int rt = pthread_create(&m_thread, nullptr, &Thread::run, this);
    if (rt) {
        SYLAR_LOG_ERROR(g_logger) << "pthread_create thread fail, rt=" << rt
                                  << " name=" << name;
        throw std::logic_error("pthread_create error");
    }
    m_semaphore.wait();
}
extern int pthread_create (pthread_t *__restrict __newthread,
			   const pthread_attr_t *__restrict __attr,
			   void *(*__start_routine) (void *),
			   void *__restrict __arg) __THROWNL __nonnull ((1, 3));
```

可以看到主要是对函数`pthread_create`的封装。这个函数其实前一段时间自己刚好看过。第一个参数为指向`pthread_t`类型的指针，用于返回新线程的ID。第二个参数为指向`pthread_attr_t`类型的指针，该结构体包含一些有关新线程属性的信息。可以将其设置为NULL以使用默认值。最值得讨论的是后面两个参数，第三个参数为后调函数，用一个void指针指向。当执行`pthread_create`之后创建一个新线程之后，就开始执行这个后调函数。当然这个函数可能会有参数值，第四个参数即为后调函数的参数值。估计是兼容性的考虑，这个同样是用了void指针来表示，这样就可以传递任意个数的任意数据类型。

析构函数：

```c++
Thread::~Thread() {
    if (m_thread) {
        pthread_detach(m_thread);
    }
}
extern int pthread_detach (pthread_t __th) __THROW;
```

首先检测此线程是否存在，如果存在，则调用`pthread_detach`函数释放线程关联资源。

join（等待线程执行完成）

```c++
void Thread::join() {
    if (m_thread) {
        int rt = pthread_join(m_thread, nullptr);
        if (rt) {
            SYLAR_LOG_ERROR(g_logger) << "pthread_join thread fail, rt=" << rt
                                      << " name=" << m_name;
            throw std::logic_error("pthread_join error");
        }
        m_thread = 0;
    }
}
extern int pthread_join (pthread_t __th, void **__thread_return);
```

用于等待指定线程的终止(即参数m_thread)，并获得该线程的返回值。后面用于储存线程返回的值。如果不需要获取返回值，则可以将其设置为NULL。显然是一个线程来调用这个函数，调用之后这个线程等待指定线程的结束，相当于阻塞了，等待获取返回值并回收线程所使用的资源。

run（线程执行函数）

```c++
void *Thread::run(void *arg) {
    Thread *thread = (Thread *)arg;
    t_thread       = thread;
    t_thread_name  = thread->m_name;
    thread->m_id   = sylar::GetThreadId();
    pthread_setname_np(pthread_self(), thread->m_name.substr(0, 15).c_str());

    std::function<void()> cb;
    cb.swap(thread->m_cb);

    thread->m_semaphore.notify();

    cb();
    return 0;
}
```

>通过信号量，能够确保构造函数在创建线程之后会一直阻塞，直到`run`方法运行并通知信号量，构造函数才会返回。
>
>在构造函数中完成线程的启动和初始化操作，可能会导致线程还没有完全启动就被调用，从而导致一些未知的问题。因此，在出构造函数之前，确保线程先跑起来，保证能够初始化id，可以避免这种情况的发生。同时，这也可以保证线程的安全性和稳定性。

作为线程创造函数中的回调函数，即在创建线程之后运行的函数。这里才是新线程运行的地方。



上面讲了生成线程中的回调函数，同类的还有两种用法：`std::function`&`std::bind`，这两种方法都是对回调对象进行统一和封装，这两种都是在C++11标准中新增的。

C++可调用对象有：**函数、函数指针、lambda表达式、bind对象、函数对象**。std::function是一个可调用对象包装器，是一个类模板，可以容纳除了类成员函数指针之外的所有可调用对象，它可以用统一的方式处理函数、函数对象、函数指针，并允许保存和延迟它们的执行。

```C++
std::function<函数类型>
```

一般会和typedef别名统一运用。举个例子：

```c++
typedef std::function<int(int, int)> comfun;
int add(int a, int b) { return a + b; }
int main(){
	comfun a = add;
    std::cout << a(5, 3) << std::endl;
}
```

可以看到其实就像是一个函数指针一样。

std::function有两个优点，一个是封装形成一个新的可调用的std::function对象，简化使用，另一个是比较安全(这个自己似乎很少涉及到)。

std::bind可以看作一个通用的函数适配器，**它接受一个可调用对象，生成一个新的可调用对象来适应原对象的参数列表**。

std::bind将可调用对象与其参数一起进行绑定，绑定后的结果可以使用std::function保存。std::bind主要有以下两个作用：

- **将可调用对象和其参数绑定成一个仿函数**；
- **只绑定部分参数，减少可调用对象传入的参数**。

第二个作用还挺有意思的，就是可以空出位置来，然后再输入参数调用进去。

下面我们分析`test_thread.cc`中的相关操作。

首先就是和前面三节学习最密切相关日志，命令行处理，配置文件读入三大项：

```c++
sylar::Logger::ptr g_logger = SYLAR_LOG_ROOT();
sylar::EnvMgr::GetInstance()->init(argc, argv);
    sylar::Config::LoadFromConfDir(sylar::EnvMgr::GetInstance()->getConfigPath());
```

之后的操作是循环生成三个线程，并将这三个线程添加到线程vector中去。可以看到线程生成主要是由`ylar::Thread::ptr thr(new sylar::Thread(std::bind(func1, &arg), "thread_" + std::to_string(i)));`这行代码来完成的。

观察`thread`的构造函数，其中后面的参数传入的是本线程的名称，即字符串"thread_"和当前i的值的拼接，例如在第一次循环中i=0，线程的名称则为thread_0。前一个函数为回调函数，用bind来表示。实际上回调函数是`func`，传递给他的参数是`arg`。需要注意的是，回调函数不会立刻执行，而是会先存着，之后在某个指定时刻执行。

进入到构造函数`Thread::Thread(std::function<void()> cb, const std::string &name) : m_cb(cb), m_name(name) `中，这里相当于是bind将回调函数传入了function中，然后又将此回调函数储存在`m_cb`中。可以看到在构造函数中最重要的就是线程生成函数，即`int rt = pthread_create(&m_thread, nullptr, &Thread::run, this);`。

这个函数我们之前也说过，其中`run`函数即为后调函数，在生成线程中需要运行的函数。

```c++
void *Thread::run(void *arg) {
    Thread *thread = (Thread *)arg; // 拿到新创建的Thread对象
    t_thread       = thread; // 更新当前线程
    t_thread_name  = thread->m_name;
    thread->m_id   = sylar::GetThreadId(); // 设置当前线程的id
    // 只有进了run方法才是新线程在执行，创建时是由主线程完成的，threadId为主线程的
    pthread_setname_np(pthread_self(), thread->m_name.substr(0, 15).c_str());  // 设置线程名称
    // pthread_creat时返回引用 防止函数有智能指针,
    std::function<void()> cb;
    cb.swap(thread->m_cb);
    // 在出构造函数之前，确保线程先跑起来，保证能够初始化id
    thread->m_semaphore.notify();

    cb();
    return 0;
}
```

在进入`run`函数之后，新的线程其实刚刚创建，此时新线程的id和原线程都是一致的。在gdb中调试可以看到：

![image-20240627094053882](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240627094053882.png)

之后是对于新线程的初始化，函数`pthread_setname_np`是对于线程名字的设置，运行完这个函数可以在gdb中看到：

![image-20240627094337546](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240627094337546.png)

之后的两行`std::function<void()> cb; cb.swap(thread->m_cb);`则是对于cb的初始化。我们用cb交换获得m_cb函数，即我们之前存的func1函数。之后运行`notify`，原型为`sem_post`，将信号量加1，使新生成的线程得以运行。

之后就是`cb()`，根据gdb调试可以看到，`cb()`函数进入了之前bind调入的回调函数`func1`，印证了我们之前的推测。

不过需要注意的是，由于多线程运行可能会存在多个线程同时运行的情况，所以可以用`set scheduler-locking on`命令，即之前当前进程，会使调试更加顺利一些。同时在调试的时候也可以多使用`info threads`，可以看到当前有那些线程存在，以及当前观察运行的线程是那个。

总结一下这部分的代码，我们在主线程中循环创建子线程，并将子线程添加到线程数组中去，创建的时候都是从主线程创建一个副本，这个副本一开始和主线程是一致的，之后我们进入回调函数之中，修改子线程的名字，然后再做一些操作，最终得到子线程。之后我们将子线程加入到线程数组`thrs`中去。

之后我们调用`Thread::join()`循环结束这三个子线程。需要注意的是，可能在我们调用这个函数之前子线程就结束并退出了，出现这样的字样：

![image-20240627160549056](C:\Users\Linyuan\AppData\Roaming\Typora\typora-user-images\image-20240627160549056.png)

这个输出自己没有在sylar工程中找到，大概是关于线程的库函数中输出的。多次调试，线程退出的时间也不太一致，这和之前讲的关于线程并行运行是一致的。



下面是关于锁模块的拆解。

## Semaphore（信号量）

```c++
private:
    sem_t m_semaphore; //本质是一个长整型的数
```

构造函数：

```c++
Semaphore::Semaphore(uint32_t count) {
    if (sem_init(&m_semaphore, 0, count)) {
        throw std::logic_error("sem_init error");
    }
}
extern int sem_init (sem_t *__sem, int __pshared, unsigned int __value)
  __THROW __nonnull ((1));
```

> 参数 `sem` 是指向要初始化的信号量的指针；参数 `pshared` 指定了信号量是进程内共享还是跨进程共享，如果值为 0，则表示进程内共享；参数 `value` 是信号量的初始值。该函数成功时返回 0，否则返回 -1，并设置适当的错误码。

析构函数：

```c++
Semaphore::~Semaphore() {
    sem_destroy(&m_semaphore);
}
extern int sem_destroy (sem_t *__sem) __THROW __nonnull ((1));
```

就像之前讲的一样，需要确保此信号量不被任何线程或者进程使用才能调用析构函数。否则会导致未定义的行为。

> 此外，如果在调用 `sem_destroy()` 函数之前，没有使用 `sem_post()` 函数将信号量的值增加到其初始值，则可能会导致在销毁信号量时出现死锁情况。

`wait`(获取信号量)，函数原型为`int sem_wait(sem_t *sem);`

```c++
void Semaphore::wait() {
    if(sem_wait(&m_semaphore)) {
        throw std::logic_error("sem_wait error");
    }
}
```

参数 `sem` 是指向要获取的信号量的指针。如果当前信号量的值大于0，则表示可以调用，将信号量值减1后进行使用。如果当前信号量小于等于0，则表示不能调用，被阻塞。

`notify(释放信号量)`，函数原型是`int sem_post(sem_t *sem);`向指定的命名或者未命名信号量发送信号，使信号量值加1。如果有进程活线程正在等待该信号量，则其中一个唤醒以继续执行。

```c++
void Semaphore::notify() {
    if(sem_post(&m_semaphore)) {
        throw std::logic_error("sem_post error");
    }
}
extern int sem_post (sem_t *__sem) __THROWNL __nonnull ((1));
```

### ScopedLockImpl(局部锁的模板实现)

这个相当于一个自动上锁模板，即构造的时候上锁，析构的开锁。

```c++
/**
     * @brief 构造函数
     * @param[in] mutex Mutex
     */
    ScopedLockImpl(T& mutex)
        :m_mutex(mutex) {
        m_mutex.lock();
        m_locked = true;
    }
    /**
     * @brief 析构函数,自动释放锁
     */
    ~ScopedLockImpl() {
        unlock();
    }
```

与此结构类似的，还有`ReadScopedLockImpl`&&`WriteScopedLockImpl`，用来封装读写锁。

### Mutex(互斥锁)

```c++
private: pthread_mutex_t m_mutex;
```

构造函数：

```C++
Mutex() {pthread_mutex_init(&m_mutex, nullptr);}
extern int pthread_mutex_init (pthread_mutex_t *__mutex,
			       const pthread_mutexattr_t *__mutexattr)
     __THROW __nonnull ((1));
```

`__mutex`是指向要初始化的互斥锁对象的指针。`__mutexattr`是指向互斥锁属性对象的指针，可以为NULL以使用默认属性。

析构函数：

```c++
~Mutex () {pthread_mutex_destroy(&m_mutex);}
extern int pthread_mutex_destroy (pthread_mutex_t *__mutex)
     __THROW __nonnull ((1));
```

加锁：

```c++
void lock() {pthread_mutex_lock(&m_mutex);}

extern int pthread_mutex_lock (pthread_mutex_t *__mutex)
     __THROWNL __nonnull ((1));
```

解锁：

```c++
void unlock() {pthread_mutex_unlock(&m_mutex);}
extern int pthread_mutex_unlock (pthread_mutex_t *__mutex)
     __THROWNL __nonnull ((1));
```

可以看到，这其实是一种封装。

### RWMutex（读写锁）

这个和上面的互斥锁很像，都是进行了一下封装。不过和自己想象中不太一样的是，读写锁是一个锁，可以分别上读锁，上写锁，不过解锁是用同一个函数来进行解锁的。

### Spinlock（自旋锁）

与mutex不同，自旋锁不会进入睡眠状态，而是在获取锁时进行忙等，直到锁可以用。当锁被释放时，等待获取锁的线程将立即获取锁，从而避免了线程进入和退出睡眠状态的额外开销。

构造函数：

```c++
Spinlock() {pthread_spin_init(&m_mutex, 0);}
extern int pthread_spin_init (pthread_spinlock_t *__lock, int __pshared)
     __THROW __nonnull ((1));
```

其中第一个参数`&m_mutex`表示要初始化的自旋锁变量，第二个参数`0`表示使用默认的属性。在调用`pthread_spin_init`函数之前，必须先分配内存空间来存储自旋锁变量。与`pthread_rwlock_t`类似，需要在使用自旋锁前先进行初始化才能正确使用。

分配空间以及初始化应该是在函数内部实现的，即`pthread_spin_init`中实现的。

析构函数：

```c++
~Spinlock() {pthread_spin_destroy(&m_mutex);}
extern int pthread_spin_destroy (pthread_spinlock_t *__lock)
     __THROW __nonnull ((1));
```

进行销毁自旋锁的操作。该函数确保在销毁自旋锁之前所有等待的线程都被解除阻塞并返回适当的错误码。

lock(加锁)：

```C++
void lock() {pthread_spin_lock(&m_mutex);}
extern int pthread_spin_lock (pthread_spinlock_t *__lock)
     __THROWNL __nonnull ((1));
```

`pthread_spin_lock()`用于获取自旋锁(pthread_spinlock_t)上的排他访问权限。与`mutex`不同，自旋锁在获取锁时忙等待，即不断地检查锁状态是否可用，如果不可用则一直循环等待，直到锁可用。当锁被其他线程持有时，调用`pthread_spin_lock()`的线程将在自旋等待中消耗CPU时间，直到锁被释放并获取到锁。

unlock(解锁):

```c++
void unlock() {pthread_spin_unlock(&m_mutex);}
extern int pthread_spin_unlock (pthread_spinlock_t *__lock)
     __THROWNL __nonnull ((1));
```

`pthread_spin_unlock()`用于释放自旋锁(pthread_spinlock_t)。调用该函数可以使其他线程获取相应的锁来访问共享资源。与`mutex`不同，自旋锁在释放锁时并不会导致线程进入睡眠状态，而是立即释放锁并允许等待获取锁的线程快速地获取锁来访问共享资源，从而避免了线程进入和退出睡眠状态的额外开销。

要理解自旋的时机，是在想获得，但还没有获得的时候循环检查锁，这时进行自旋，当获得之后就开始进行自己的事情了，就不需要自旋了，其他像访问的线程就开始自旋了。



### CASLock（原子锁）

成员变量与构造函数：

```c++
volatile std::atomic_flag m_mutex;
CASLock() {m_mutex.clear();}
```

volatile关键字表示该变量可能会被异步修改，因此编译器不会对其进行优化，而是每次都从内存中读取该变量的值。所以其实很多时候都是从寄存器中读取的。这个让自己想起了之前的面试mmm。

这个构造函数和之前看到不太一样。atomic_flag.clear()是C++标准库中的一个原子操作函数，用于将给定的原子标志位（atomic flag）清除或重置为未设置状态。清除之后就可以用了。

lock(上锁)：

```c++
void lock() {
        while(std::atomic_flag_test_and_set_explicit(&m_mutex, std::memory_order_acquire));
    }
inline bool
  atomic_flag_test_and_set_explicit(atomic_flag* __a,
				    memory_order __m) noexcept
  { return __a->test_and_set(__m); }    
```

`atomic_flag_test_and_set_explicit`是一个原子操作函数，包含的操作有测试给定的原子标志位atomic_flag是否被设置，并在测试之后将其设置为已设置状态。

`memory_order`是一种内存序，用于指定原子操作的徒步和内存顺序。具体来说，使用`memory_order`可以确保在当前线程获取锁之前，所有该线程之前发生的写操作都被完全同步到主内存中。这样可以防止编译器或硬件对写操作进行重排序或延迟，从而确保其他线程可以正确地读取共享资源的最新值。

unlock(解锁)：

```c++
void unlock() {
        std::atomic_flag_clear_explicit(&m_mutex, std::memory_order_release);
    }
inline void
  atomic_flag_clear_explicit(atomic_flag* __a, memory_order __m) noexcept
  { __a->clear(__m); }

```

`atomic_flag_clear_explicit()`同样是一个原子操作函数，将给定的原子标志位清除活重置位未设置状态。


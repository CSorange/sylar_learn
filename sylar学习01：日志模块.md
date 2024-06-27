# sylar学习01：日志模块

因为最终都是要落在main函数中，所以自己的学习也是从main函数中展开的，与日志有关的文件显然是```test_log.cpp```。

不过观察所有的test文件，会发现一个共识，即每个测试文件在main函数之外都会有

```c++
sylar::Logger::ptr g_logger = SYLAR_LOG_ROOT();
```

因为每个测试我们都需要和日志进行交互，这个测试运行的怎么样呢，跑的好不好呢，都需要保存到日志中。这行代码其实是对于日志的初始化。需要注意的是，sylar整个工程中用到了很多的宏定义，这会让后面的代码很简洁，但这也需要我们逐层向里分析。关于日志所需的类定义集中在```log.h```，我们会跟着```SYLAR_LOG_ROOT()```的脚步去看看都有那些结构。

### SYLAR_LOG_ROOT()

```c++
#define SYLAR_LOG_ROOT() sylar::LoggerMgr::GetInstance()->getRoot()
```

首先是最外层的日志管理器：`LoggerManager`，通过`LoggerMgr`来获得`logger`。这里涉及到一个概念，即单例模式智能指针封装类，定义在`singleton.h`文件中。

> 关于单例模式：
>
> 意图：确保一个类只有一个实例，并提供一个全局访问点来访问该实例。
>
> 主要解决：频繁创建和销毁全局使用的类实例的问题。
>
> 何时使用：当需要控制实例数目，节省系统资源时。
>
> 如何解决：检查系统是否已经存在该单例，如果存在则返回该实例；如果不存在则创建一个新实例。
>
> 关键代码：构造函数是私有的。
>
> 优点：
>
> - 内存中只有一个实例，减少内存开销，尤其是频繁创建和销毁实例时（如管理学院首页页面缓存）。
> - 避免资源的多重占用（如写文件操作）。
>
> 缺点：
>
> - 没有接口，不能继承。
> - 与单一职责原则冲突，一个类应该只关心内部逻辑，而不关心实例化方式。

单例模式智能指针封装类显然又使用了智能指针。

>日志管理使用单例模式，保证从容器`m_loggers`中拿出的日志器是唯一不变的。日志管理器会初始化一个主日志器放到容器中，若再创建一个新的日志器时没有设置`appender`，则会使用这个主日志器进行日志的输出，但是输出时日志名称并不是主日志器的，因为在输出时是按照`event`中的`logger`的名称输出的。

`getRoot()`返回值为一个智能指针，即root日志器。在此之前，我们需要先初始化`LoggerManager`。下图为`LoggerManager`的构造函数。

```c++
LoggerManager::LoggerManager() {
    m_root.reset(new Logger("root"));
    m_root->addAppender(LogAppender::ptr(new StdoutLogAppender));
    m_loggers[m_root->getName()] = m_root;
    init();
}
```

1. 初始化了`logger`，即日志类。

```c++
Logger::Logger(const std::string &name)
    : m_name(name)
    , m_level(LogLevel::INFO)
    , m_createTime(GetElapsedMS()) {
    }
```

`logger`初始化了日志类的名字(root)，等级(INFO)，以及标注了创建时间。

2. 初始化一个输出到控制台的Appender并添加。StdoutLogAppender的基类为LogAppender，LogAppender可以派生出两种类，一种是输出到控制台的，一种是输出到列文件的。

   下面为StdoutLogAppender的构造函数。

```c++
StdoutLogAppender::StdoutLogAppender()
    : LogAppender(LogFormatter::ptr(new LogFormatter)) {}
```

​		可以看到初始化了`LogFormatter`，为日志格式化。下面为LogFormatter的构造函数。

```c++
LogFormatter::LogFormatter(const std::string &pattern)
    : m_pattern(pattern) {
    init();
}
```

​		其中`init()`函数为提取pattern中的常规字符和模式字符。



### 其他类

除了上面涉及到的类以外，还有一些类。

`LogLevel`：日志等级，我们会给日志赋予一个等级，优先级越高，则越需要被优先处理。

`LogEvent`：日志事件，主要是用来输出一些东西。

`LogEventWrap`：日志事件包装器，只是为了方便宏定义，内部包含日志事件和日志器。



#### 将不同日志级别的事件写入logger中

`test_log.cpp`文件主要检验的是事件写入logger对应的两组函数，分别为：`**SYLAR_LOG_LEVEL**(logger, level)`和`SYLAR_LOG_FMT_LEVEL(logger, level, fmt, ...)`，前者使用流的方式，后者使用格式化的方式。

```c
#define SYLAR_LOG_LEVEL(logger , level) \
    if(level <= logger->getLevel()) \
        sylar::LogEventWrap(logger, sylar::LogEvent::ptr(new sylar::LogEvent(logger->getName(), \
            level, __FILE__, __LINE__, sylar::GetElapsedMS() - logger->getCreateTime(), \
            sylar::GetThreadId(), sylar::GetFiberId(), time(0), sylar::GetThreadName()))).getLogEvent()->getSS()
```

```
#define SYLAR_LOG_FMT_LEVEL(logger, level, fmt, ...) \
    if(level <= logger->getLevel()) \
        sylar::LogEventWrap(logger, sylar::LogEvent::ptr(new sylar::LogEvent(logger->getName(), \
            level, __FILE__, __LINE__, sylar::GetElapsedMS() - logger->getCreateTime(), \
            sylar::GetThreadId(), sylar::GetFiberId(), time(0), sylar::GetThreadName()))).getLogEvent()->printf(fmt, __VA_ARGS__)
```

两者其实就是最后一点有所区别，一个是使用getSS()函数获取字节流，另一个是使用printf进行输出。



在这里我们初始化`LogEventWrap`。上面说到，`LogEventWrap`包含日志事件和日志器，其中logger为日志器，后面为日志事件。

观察日志事件的构造函数：

```c++
LogEvent::LogEvent(const std::string &logger_name, LogLevel::Level level, const char *file, int32_t line
        , int64_t elapse, uint32_t thread_id, uint64_t fiber_id, time_t time
        , const std::string &thread_name)
    : m_level(level)
    , m_file(file)
    , m_line(line)
    , m_elapse(elapse)
    , m_threadId(thread_id)
    , m_fiberId(fiber_id)
    , m_time(time)
    , m_threadName(thread_name)
    , m_loggerName(logger_name) {
}
```

为什么使用这个类`LogEventWrap`：当想使用该宏打一次日志后，由于LogEvent使用的是智能指针，在定义该宏的作用域下这个`LogEvent`并不会立即释放，所以使用`LogEventWarp`包装`LogEvent::ptr`当定义该宏的语句执行完后就会自动进入析构函数，并将`LogEvent`写入`Logger`中,打印出日志。

构造之后，调用析构函数`~LogEventWrap()`

```C++
LogEventWrap::~LogEventWrap() {
    m_logger->log(m_event);
}
```

```c++
//调用Logger的所有appenders将日志写一遍
void StdoutLogAppender::log(LogEvent::ptr event) {
    if(m_formatter) {
        m_formatter->format(std::cout, event);
    } else {
        m_defaultFormatter->format(std::cout, event);
    }
}
```

输出的结果举例：2023-04-26 15:12:12 3613 iom 0 [INFO] [root] tests/test_hook.cc:69 hello world.


# sylar学习02：环境变量模块

接着看`test_log.cpp`文件，进入`main`函数之后首先要做的是：

```c++
sylar::EnvMgr::GetInstance()->init(argc, argv);
    sylar::Config::LoadFromConfDir(sylar::EnvMgr::GetInstance()->getConfigPath());
```

其中第一行处理环境变量模块，第二行处理配置模块，我们首先来看环境变量模块。

同样是熟悉的单例，与环境变量相关的类只有一个`Env`，所含有的参数为：

```c++
private:
    /// Mutex
    RWMutexType m_mutex;
    /// 存储程序的自定义环境变量
    std::map<std::string, std::string> m_args;
    /// 存储帮助选项与描述，就是-h之后会显示的信息
    std::vector<std::pair<std::string, std::string>> m_helps;

    /// 程序名，也就是argv[0]，这两个竟然是等价了，自己倒不是很清楚
    std::string m_program;
    /// 程序完整路径名，也就是/proc/$pid/exe软链接指定的路径 
    std::string m_exe;
    /// 当前路径，从argv[0]中获取
    std::string m_cwd;
```

当初始化一个`Env`类时，首先需要进行的是初始化，即`init()`。此函数将`main`函数的参数原样传入`init`接口中，以便于从`main`函数参数中提取命令行选项与值，以及通过`argv[0]`参数获取命令行程序名称，更通俗一点来说就是将输入参数的那一行转换为参数数组。

其他提供的一些接口函数：

1. add/get/has/del：用于操作程序自定义环境变量，参数为key-value，get操作支持传入默认值，在对应的环境变量不存在时，返回这个默认值。
2. setEnv/getEnv: 用于操作系统环境变量，对应标准库的setenv/getenv操作。
3. addHelp/removeHelp/printHelp: 用于操作帮助选项和描述信息。
4. getExe/getCwd/getAbsolutePath: 用于获取程序名称，程序路径，绝对路径。
5. getConfigPath: 获取配置文件夹路径，配置文件夹路径由命令行-c选项传入。

目前来看`help`中也没有什么提示，所以最主要就是配置文件所在的位置，或者说只有这个。


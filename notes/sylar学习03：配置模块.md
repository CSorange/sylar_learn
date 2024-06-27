# sylar学习03：配置模块

歪个楼，先说一下回调函数。自己在之前生成线程的函数中也见到了。

>**回调函数**是作为参数传递到另一个函数中，然后在外部函数内调用以完成某种例行程序或操作的函数。
>
>基于回调的 API 的使用者需要编写一个被传递到 API 中的函数。API 的提供者（称为*调用方*）接受该函数，并在调用方的主体内的某个时刻回调（或者说，执行）该函数。调用方负责将正确的参数传递给回调函数。调用方也可能期望从回调函数中获得特定的返回值，用于指示调用方的进一步行为。



这里的配置文件是使用YAML格式，即我们经常见到的.yml后缀文件。这个和json是并列的关系，即是两种文件格式。YAML支持以下集中数据类型：对象(Map)，数组(Sequence)，纯量(scalars)。

配置模块里主要有三个类：

1. **class ConfigVarBase**：配置变量的基类
2. **class ConfigVar**：配置参数模板子类，保存对应类型的参数值，通过仿函数实现`string`和`T类型`之间的相互转化
3. **class Config**： `ConfigVar`的管理类

### `ConfigVarBase`

`ConfigVarBase`提供三个纯虚函数(toString、formString、getTypeName)供子类`ConfigVar`实现。

```c++
protected:
    /// 配置参数的名称
    std::string m_name;
    /// 配置参数的描述
    std::string m_description;
```

相当于只有一些基础的东西，如名称以及描述。

```c++
ConfigVarBase(const std::string &name, const std::string &description = "")
        : m_name(name)
        , m_description(description) {
        std::transform(m_name.begin(), m_name.end(), m_name.begin(), ::tolower); //将大写转化为小写，所以其实不需要区分大小写
    }
```

需要注意的是，构造函数中除了赋值以外还有一个transform函数，这个函数是将文本字母从大写转化为小写，即都是文本内容均是大小写不敏感类型。

在定义`ConfigVar`之前先构造了几个模板类`LexicalCast`，这个讲述了一些类型强制转化，包括字符串和vector、list、set、unordered_set、map、unordered_map之间相互的转化。

### `ConfigVar`(ConfigVar管理类)

```c++
template <class T, class FromStr = LexicalCast<std::string, T>, class ToStr = LexicalCast<T, std::string>>
class ConfigVar : public ConfigVarBase {}
```

配置参数中的`class FromStr = LexicalCast<std::string, T>`和`class ToStr = LexicalCast<T, std::string>`即对应虚函数中需要实现的formString和toString。

```c++
ConfigVar(const std::string &name, const T &default_value, const std::string &description = "")
        : ConfigVarBase(name, description)
        , m_val(default_value) {
    }
```

其实就是回到`ConfigVarBase`中去。

获取和设置参数：

```c++
/**
     * @brief 获取当前参数的值
     */
    const T getValue() {
        RWMutexType::ReadLock lock(m_mutex);
        return m_val;
    }

    /**
     * @brief 设置当前参数的值
     * @details 如果参数的值有发生变化,则通知对应的注册回调函数
     * 这里就是很典型的锁应用，先获取读锁进行读，然后如果不一样需要写回，则获取写锁进行写
     */
    void setValue(const T &v) {
        {
            RWMutexType::ReadLock lock(m_mutex);
            if (v == m_val) {
                return;
            }
            for (auto &i : m_cbs) {
                i.second(m_val, v);
            }
        }
        RWMutexType::WriteLock lock(m_mutex);
        m_val = v;
    }
```

在`setValue`函数中，如果v与m_val的值不同，则需要回调变更函数。

回调函数`addListener`：

```c++
uint64_t addListener(on_change_cb cb) {
    static uint64_t s_fun_id = 0;
    RWMutexType::WriteLock lock(m_mutex);
    ++s_fun_id;
    m_cbs[s_fun_id] = cb;
​
    return s_fun_id;
}
```

关于回调函数用到的map：

```c++
std::map<uint64_t, on_change_cb> m_cbs;
typedef std::function<void(const T &old_value, const T &new_value)> on_change_cb;
```

关于回调函数维基百科说：

> 定义：**回调函数**或简称**回调**（callback），是指对某一段[可执行代码](https://zh.wikipedia.org/wiki/可执行文件)的[引用](https://zh.wikipedia.org/wiki/引用_(程序设计))，它被作为[参数](https://zh.wikipedia.org/wiki/參數_(程式設計))传递给另一段代码；预期这段代码将回调（执行）这个回调函数作为自己工作的一部分。
>
> 使用1：假设有一个函数，其功能为读取配置文件并由文件内容设置对应的选项。若这些选项由[散列值](https://zh.wikipedia.org/wiki/散列函数)所标记，则让这个函数接受一个回调会使得程序设计更加灵活：函数的调用者可以使用所希望的散列算法，该算法由一个将选项名转变为散列值的回调函数实现；因此，回调允许函数调用者在运行时调整原始函数的行为。
>
> 使用2：回调的另一种用途在于处理信号或者类似物。例如一个[POSIX](https://zh.wikipedia.org/wiki/POSIX)程序可能在收到[SIGTERM](https://zh.wikipedia.org/wiki/SIGTERM)信号时不愿立即终止；为了保证一切运行良好，该程序可以将清理函数注册为SIGTERM信号对应的回调。

这里用的是使用1。

### `Config`（`ConfigVar`管理类）

```C++
typedef std::unordered_map<std::string, ConfigVarBase::ptr> ConfigVarMap; //所有的配置项
typedef RWMutex RWMutexType; //配置项的RWMutex
```

后面主要函数为`LoadFromConfDir`，即加载文件夹中的配置文件。

```c++
/**
     * @brief 加载path文件夹里面的配置文件
     */
    static void LoadFromConfDir(const std::string &path, bool force = false);
```



```c++
void Config::LoadFromConfDir(const std::string &path, bool force) {
    std::string absoulte_path = sylar::EnvMgr::GetInstance()->getAbsolutePath(path);
    std::vector<std::string> files;
    FSUtil::ListAllFile(files, absoulte_path, ".yml");
    //一般会定义到路径文件夹中带有.yml的文件
    for (auto &i : files) {
        {
            struct stat st;
            lstat(i.c_str(), &st);
            sylar::Mutex::Lock lock(s_mutex);
            if (!force && s_file2modifytime[i] == (uint64_t)st.st_mtime) {
                continue;
            }
            s_file2modifytime[i] = st.st_mtime; //记录文件修改时间
        }
        try {
            YAML::Node root = YAML::LoadFile(i);
            LoadFromYaml(root); //模块初始化配置
            SYLAR_LOG_INFO(g_logger) << "LoadConfFile file="
                                     << i << " ok";
        } catch (...) {
            SYLAR_LOG_ERROR(g_logger) << "LoadConfFile file="
                                      << i << " failed";
        }
    }
}
```

这个函数也是test文件中用来配置的。首先会确认文件夹的具体路径，以及配置文件的后缀，比如这里就是path文件夹以及以.yml为后缀的名字。当我们锁定目标文件之后，就会需要遍历每一个文件来进行配置。首先我们会记录文件修改时间，其次我们进入`LoadFromYaml`函数。我们将一个模块需要初始化的内容单独放在了一个.yml文件里。

我们将root中的所有的节点信息保存倒all_nodes中。在函数`ListAllMember`中，我们使用递归的方式遍历YAML格式的配置文件中的所有成员，将每个节点的名称和值存在list中。

我们以`test_config.yml`文件为例，文件内容为：

```
global:
    int: 8888
    float: 3.14
    string: brand new name
    int_vec: [4, 5, 6]
    int_list: [4, 5, 6]
    int_set: [4, 5, 6]
    int_unordered_set: [4, 5, 6]
    map_string2int:
        key1: 3
        key2: 4
    unordered_map_string2int:
        key1: 3
        key2: 4
```

存在all_nodes中的模样为：

```c++
std::__cxx11::list = {[0] = {first = "", second = {m_isValid = true, m_invalidKey = "", 
      m_pMemory = std::shared_ptr<YAML::detail::memory_holder> (use count 33, weak count 0) = {
        get() = 0x55555575f240}, m_pNode = 0x555555764370}}, [1] = {first = "global", second = {
      m_isValid = true, m_invalidKey = "", 
      m_pMemory = std::shared_ptr<YAML::detail::memory_holder> (use count 33, weak count 0) = {
        get() = 0x55555575f240}, m_pNode = 0x555555765160}}, [2] = {first = "global.int", second = {
      m_isValid = true, m_invalidKey = "", 
      m_pMemory = std::shared_ptr<YAML::detail::memory_holder> (use count 33, weak count 0) = {
        get() = 0x55555575f240}, m_pNode = 0x555555764280}}, [3] = {first = "global.float", second = {
      m_isValid = true, m_invalidKey = "",
```

我们保存下来每一行，格式即根据缩进进行嵌套。


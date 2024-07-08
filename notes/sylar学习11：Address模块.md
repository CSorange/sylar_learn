# sylar学习11：Address模块

`socket`的通信都需要与地址打交道，所以封装了`Address`模块，不用进行繁琐的一系列操作。

> `Address`模块有主要一下几类：
>
> - **class Address**：基类，抽象类，对应`sockaddr`类型。提供获取地址的方法，以及一些基础操作的纯虚函数。
> - **class IPAddress**：继承`Address`，抽象类。提供关于`IP`操作的纯虚函数。
> - **class IPv4Address**：继承`IPAddress`，对应`sockaddr_in`类型。一个`IPv4`地址。
> - **class IPv6Address**：继承`IPAddress`，对应`sockaddr_in6`类型。一个`IPv6`地址。
> - **class UinxAddress**：继承`Address`，`Unix`域套接字类，对应`sockaddr_un`类型。一个`Unix`地址。
> - **class UnknowAddress**：继承`Address`，对应`sockaddr`类型。未知地址。

socket地址结构体：`sockaddr`和`sockaddr_in`都为16字节，`sockaddr_in6`为28字节。

尽管字节数不同，但依然可以相互转化，因为在进行类型转换时只需要将指针类型进行转换即可，不需要改变结构体的大小。

```c++
/* Structure describing a generic socket address.  */
struct sockaddr
{
 uint16 sa_family;           /* Common data: address family and length.  */
 char sa_data[14];           /* Address data.  */
};

/* Structure describing an Internet socket address.  */
struct sockaddr_in
{
 uint16 sin_family;          /* Address family AF_INET */ 
 uint16 sin_port;            /* Port number.  */
 uint32 sin_addr.s_addr;     /* Internet address.  */
 unsigned char sin_zero[8];  /* Pad to size of `struct sockaddr'.  */
};

/* Ditto, for IPv6.  */
struct sockaddr_in6
{
 uint16 sin6_family;         /* Address family AF_INET6 */
 uint16 sin6_port;           /* Transport layer port # */
 uint32 sin6_flowinfo;       /* IPv6 flow information */
 uint8  sin6_addr[16];       /* IPv6 address */
 uint32 sin6_scope_id;       /* IPv6 scope-id */
};

struct in6_addr
  {
    union
      {
    uint8_t __u6_addr8[16];
    uint16_t __u6_addr16[8];
    uint32_t __u6_addr32[4];
      } __in6_u;
#define s6_addr         __in6_u.__u6_addr8
#ifdef __USE_MISC
# define s6_addr16      __in6_u.__u6_addr16
# define s6_addr32      __in6_u.__u6_addr32
#endif
  };
#endif /*
```

应用程序无需处理主机名，主机地址，服务名和端口字符串，我们可以用函数`getaddrinfo`将这些转化为套接字地址结构体，即`addrinfo `。

相关的函数：

```c++
int getaddrinfo( const char *node, 
                 const char *service,
                 const struct addrinfo *hints,
                 struct addrinfo **res);
1）nodename:节点名可以是主机名，也可以是数字地址。（IPV4的10进点分，或是IPV6的16进制）；
2）servname:包含十进制数的端口号或服务名如（ftp,http）；
3）hints:是一个空指针或指向一个addrinfo结构的指针，由调用者填写关于它所想返回的信息类型的线索；
4）res:存放返回addrinfo结构链表的指针；

/*  释放addrinfo指针    */
void freeaddrinfo(struct addrinfo *res);
/*  getaddrinfo函数返回的错误码转换为对应的错误信息字符串    */
const char *gai_strerror(int errcode);
 
struct addrinfo {
      int              ai_flags;        // 位掩码，修改默认行为
      int              ai_family;       // socket()的第一个参数
      int              ai_socktype;     // socket()的第二个参数
      int              ai_protocol;     // socket()的第三个参数
      socklen_t        ai_addrlen;      // sizeof(ai_addr)
      struct sockaddr  *ai_addr;        // sockaddr指针
      char            *ai_canonname;    // 主机的规范名字
      struct addrinfo   *ai_next;       // 链表的下一个结点
}

参数说明:
ai_flags:
1）AI_ADDRCONFIG: 只有当本地主机被配置为IPv4时，getaddrinfo返回IPv4地址，IPv6同理。
2）AI_CANONNAME: ai_canonname默认为NULL，设置此标志位，告诉getaddrinfo将列表中第一个addrinfo结构体的ai_cannoname字段指向host的权威(官方)名字。
3）AI_NUMERICSERV: service默认为服务名或端口号。这个标志强制参数service为端口号。
4）AI_PASSIVE: getaddrinfo默认返回套接字地址，客户端可以在调用connect时用作主动套接字。此标志位告诉该函数，返回的套接字地址可能被服务器用作监听套接字。此时，host应该为NULL。
5）AI_NUMERICHOST：用于指示getaddrinfo函数在解析主机名时是否进行名称解析。当我们需要使用IP地址而不是主机名来创建套接字时，可以将AI_NUMERICHOST常量作为getaddrinfo函数的hints参数中的ai_flags成员的值，以指示getaddrinfo函数不进行主机名解析，而直接使用传入的IP地址。这样可以避免主机名解析带来的延迟和不确定性，提高套接字的创建效率和可靠性。
```

`getifaddrs`函数用于获取系统中所有网络接口的信息。

```c++
/*  getifaddrs创建一个链表，链表上的每个节点都是一个struct ifaddrs结构，getifaddrs()返回链表第一个元素的指针。
 *  成功返回0, 失败返回-1,同时errno会被赋允相应错误码。 */
int getifaddrs(struct ifaddrs **ifap);

/*  释放ifaddrs */
void freeifaddrs(struct ifaddrs *ifa);

struct ifaddrs {
   struct ifaddrs  *ifa_next;    /* 指向链表中下一个struct ifaddr结构 */
   char            *ifa_name;    /* 网络接口名 */
   unsigned int     ifa_flags;   /* 网络接口标志 */
   struct sockaddr *ifa_addr;    /* 指向一个包含网络地址的sockaddr结构 */
   struct sockaddr *ifa_netmask; /* 指向一个包含网络掩码的结构 */
   union {
       struct sockaddr *ifu_broadaddr;
                        /* 如果(ifa_flags&IFF_BROADCAST)有效，ifu_broadaddr指向一个包含广播地址的结构 */
       struct sockaddr *ifu_dstaddr;
                        /* 如果(ifa_flags&IFF_POINTOPOINT)有效，ifu_dstaddr指向一个包含p2p目的地址的结构 */
   } ifa_ifu;
#define              ifa_broadaddr ifa_ifu.ifu_broadaddr
#define              ifa_dstaddr   ifa_ifu.ifu_dstaddr
   void            *ifa_data;    /* 指向一个缓冲区，其中包含地址族私有数据。没有私有数据则为NULL */
};
```

综合来看上面的结构体，其实就是两类。一类是socket地址结构体，无论是基础的sockaddr还是涉及到网络的sockaddr_in和sockaddr_in6，里面主要的内容就是Address family，地址和端口。另一类是串起来的链表结构，一个是addrinfo，另一个是ifaddrs，前者处理的是内部结构，后者处理的是系统中所有网络接口的信息。

所以根本还是socket地址结构体，后面两者其实包含这个了，不要被带跑mmm。

关于掩码：

```c++
/*
 *  sizeof(T) * 8: 算出有多少位
 *  sizeof(T) * 8 - bits: 算出掩码0的个数
 *  1 <<: 将 1 左移0的个数位
 *  -1: 前bits位数置为0，后面的 sizeof(T) * 8 - bits 位都为1
 *  ~: 将前bits位置为1，后面全部置为0
 */
template<class T>
static T CreateMask(uint32_t bits) {
    return ~((1 << (sizeof(T) * 8 - bits)) - 1);
}

template<class T>
static uint32_t CountBytes(T value) {
    uint32_t result = 0;
    for (; value; ++result) {
        // 将最右边的1置为0
        value &= value - 1;
    }
    return result;
}
```

CreateMask就是制造掩码，比如标准长度为8位，有一个数位101，那么掩码为11111000，即0的位数和这个数的位数相同，然后前补1补到8位。

上面介绍的那些，其实都是库函数中的，更靠近底层一些。下面介绍最一开始说涉及到类。

## *class* Address

```C++
// 获得sockaddr指针
virtual const sockaddr* getAddr() const = 0;
// 获得sockaddr长度
virtual socklen_t getAddrLen() const = 0;
// 可读性输出地址
virtual std::ostream& insert(std::ostream& os) const = 0;
```

可以看到，这三个是纯虚函数，需要之后进行实现。

Create（通过sockaddr创建地址）

```c++
Address::ptr Address::Create(const sockaddr *addr, socklen_t addrlen) {
    if (addr == nullptr) {
        return nullptr;
    }

    Address::ptr result;
    switch (addr->sa_family) {
    case AF_INET:
        result.reset(new IPv4Address(*(const sockaddr_in *)addr));
        break;
    case AF_INET6:
        result.reset(new IPv6Address(*(const sockaddr_in6 *)addr));
        break;
    default:
        result.reset(new UnknownAddress(*addr));
        break;
    }
    return result;
}
```

最终创建的地址有三种可能，IPv4、IPv6或者Unknown。

Lookup（通过host地址返回所有Address）

```c++
bool Address::Lookup(std::vector<Address::ptr> &result, const std::string &host,
                     int family, int type, int protocol) {
    addrinfo hints, *results, *next;
    hints.ai_flags     = 0;
    hints.ai_family    = family;
    hints.ai_socktype  = type;
    hints.ai_protocol  = protocol;
    hints.ai_addrlen   = 0;
    hints.ai_canonname = NULL;
    hints.ai_addr      = NULL;
    hints.ai_next      = NULL;

    std::string node;
    const char *service = NULL;
    //  host = www.baidu.com:http
    //检查 ipv6 address serivce [address]: IPv6的格式是这样的
    if (!host.empty() && host[0] == '[') {
        const char *endipv6 = (const char *)memchr(host.c_str() + 1, ']', host.size() - 1);
        if (endipv6) {// 找到了 ]
            //TODO check out of range
            if (*(endipv6 + 1) == ':') {  //  是否为 ：
                service = endipv6 + 2;//  endipv6后两个字节为端口号
            }
            node = host.substr(1, endipv6 - host.c_str() - 1);//  地址为[]里的内容
        }
    }
    //  ipv4    ip:port
    //检查 node serivce
    if (node.empty()) {
        service = (const char *)memchr(host.c_str(), ':', host.size());// 找到第一个:
        if (service) {// 找到了
            if (!memchr(service + 1, ':', host.c_str() + host.size() - service - 1)) {// 后面没有 : 了
                node = host.substr(0, service - host.c_str());// 拿到地址
                ++service;// ：后面就是端口号
            }
        }
    }

    if (node.empty()) {// 如果没设置端口号，就将host赋给他
        node = host;
    }
    int error = getaddrinfo(node.c_str(), service, &hints, &results);// 获得地址链表
    if (error) {
        SYLAR_LOG_DEBUG(g_logger) << "Address::Lookup getaddress(" << host << ", "
                                  << family << ", " << type << ") err=" << error << " errstr="
                                  << gai_strerror(error);
        return false;
    }
    // results指向头节点，用next遍历
    next = results;
    while (next) {
        result.push_back(Create(next->ai_addr, (socklen_t)next->ai_addrlen));// 将得到的地址创建出来放到result容器中
        /// 一个ip/端口可以对应多种接字类型，比如SOCK_STREAM, SOCK_DGRAM, SOCK_RAW，所以这里会返回重复的结果
        SYLAR_LOG_DEBUG(g_logger) << "family:" << next->ai_family << ", sock type:" << next->ai_socktype;
        next = next->ai_next;
    }
    // 释放addrinfo指针
    freeaddrinfo(results);
    return !result.empty();
}
```

LookupAny（通过host返回任意Address）

```c++
Address::ptr Address::LookupAny(const std::string &host,
                                int family, int type, int protocol) {
    std::vector<Address::ptr> result;
    if (Lookup(result, host, family, type, protocol)) {
        return result[0];
    }
    return nullptr;
}
```

这里直接返回result[0]，可能这就是所说的任意返回么。

LookupAnyIPAddress（通过host返回任意IPAddress）

```
IPAddress::ptr Address::LookupAnyIPAddress(const std::string &host,int family, int type, int protocol) {
    std::vector<Address::ptr> result;
    if (Lookup(result, host, family, type, protocol)) {
        //for(auto& i : result) {
        //    std::cout << i->toString() << std::endl;
        //}
        for (auto &i : result) {
            IPAddress::ptr v = std::dynamic_pointer_cast<IPAddress>(i);
            if (v) {
                return v;
            }
        }
    }
    return nullptr;
}
```

可以看到这次没有直接返回第一个，而是经过了筛选。

GetInterfaceAddresses(返回本机所有网卡的<网卡名, 地址, 子网掩码位数>)

```c++
bool Address::GetInterfaceAddresses(std::vector<std::pair<Address::ptr, uint32_t>> &result, const std::string &iface, int family) {
    if (iface.empty() || iface == "*") {
        if (family == AF_INET || family == AF_UNSPEC) {//创建监听任意IP地址的连接请求的ipv4
            result.push_back(std::make_pair(Address::ptr(new IPv4Address()), 0u));
        }
        if (family == AF_INET6 || family == AF_UNSPEC) {//创建监听任意IP地址的连接请求的ipv6
            result.push_back(std::make_pair(Address::ptr(new IPv6Address()), 0u));
        }
        return true;
    }

    std::multimap<std::string, std::pair<Address::ptr, uint32_t>> results;

    if (!GetInterfaceAddresses(results, family)) {// 获取失败
        return false;
    }

    auto its = results.equal_range(iface);// 返回pair，first为第一个等于的迭代器，second为第一个大于的迭代器
    for (; its.first != its.second; ++its.first) {// 将first与second之间的全部拿出来
        result.push_back(its.first->second);
    }
    return !result.empty();
}
```

这里其实利用了上一个同名函数。另外至于用multimap这个STL的原因，是因为想用equal_range函数找出一致的一段。

getFamily（返回协议簇）

```c++
int Address::getFamily() const {
    return getAddr()->sa_family;
}
```

toString（返回可读性字符串）

```c++
std::string Address::toString() const {
    std::stringstream ss;
    insert(ss);
    return ss.str();
}
```

## *class* IPAddress

这里提供5个纯虚函数，之后会分支出IPv4和IPv6。除了这个五个纯虚函数，就只剩下一个create函数了。

```c++
// 获得广播地址
virtual IPAddress::ptr broadcastAddress(uint32_t prefix_len) = 0;
// 获得网络地址
virtual IPAddress::ptr networkAddress(uint32_t prefix_len) = 0;
// 获得子网掩码地址
virtual IPAddress::ptr subnetMask(uint32_t prefix_len) = 0;
// 获得端口号
virtual uint16_t getPort() const = 0;
// 设置端口号
virtual void setPort(uint16_t v) = 0;
```

Create（通过IP、域名、服务器名,端口号创建IPAddress）

```c++
IPAddress::ptr IPAddress::Create(const char *address, uint16_t port) {
    addrinfo hints, *results;
    memset(&hints, 0, sizeof(addrinfo));

    hints.ai_flags  = AI_NUMERICHOST;
    hints.ai_family = AF_UNSPEC;// 协议无关

    int error = getaddrinfo(address, NULL, &hints, &results);
    if (error) {
        SYLAR_LOG_DEBUG(g_logger) << "IPAddress::Create(" << address
                                  << ", " << port << ") error=" << error
                                  << " errno=" << errno << " errstr=" << strerror(errno);
        return nullptr;
    }

    try {
        IPAddress::ptr result = std::dynamic_pointer_cast<IPAddress>(
            Address::Create(results->ai_addr, (socklen_t)results->ai_addrlen));
        if (result) {
            result->setPort(port);
        }
        freeaddrinfo(results);
        return result;
    } catch (...) {
        freeaddrinfo(results);
        return nullptr;
    }
}
```

## *class* IPv4Address

成员函数：`sockaddr_in m_addr;`

```c++
IPv4Address::ptr IPv4Address::Create(const char *address, uint16_t port) {
    IPv4Address::ptr rt(new IPv4Address);
    rt->m_addr.sin_port = byteswapOnLittleEndian(port);
    int result          = inet_pton(AF_INET, address, &rt->m_addr.sin_addr);// 将一个IP地址的字符串表示转换为网络字节序的二进制形式
    if (result <= 0) {
        SYLAR_LOG_DEBUG(g_logger) << "IPv4Address::Create(" << address << ", "
                                  << port << ") rt=" << result << " errno=" << errno
                                  << " errstr=" << strerror(errno);
        return nullptr;
    }
    return rt;
}
```

IPv4Address（构造函数）

```c++
IPv4Address::IPv4Address(const sockaddr_in &address) {
    m_addr = address;
}

IPv4Address::IPv4Address(uint32_t address, uint16_t port) {
    memset(&m_addr, 0, sizeof(m_addr));
    m_addr.sin_family      = AF_INET;
    m_addr.sin_port        = byteswapOnLittleEndian(port);
    m_addr.sin_addr.s_addr = byteswapOnLittleEndian(address);
}
```

两种构造函数。

getAddr（返回sockaddr指针）

```c++
sockaddr *IPv4Address::getAddr() {
    return (sockaddr *)&m_addr;
}
```

getAddrLen（返回sockaddr的长度）

```c++
const sockaddr *IPv4Address::getAddr() const {
    return (sockaddr *)&m_addr;
}
```

insert（可读性输出地址）

```c++
std::ostream &IPv4Address::insert(std::ostream &os) const {
    uint32_t addr = byteswapOnLittleEndian(m_addr.sin_addr.s_addr);
    os << ((addr >> 24) & 0xff) << "."
       << ((addr >> 16) & 0xff) << "."
       << ((addr >> 8) & 0xff) << "."
       << (addr & 0xff);
    os << ":" << byteswapOnLittleEndian(m_addr.sin_port);
    return os;
}
```

这个就是标准输出，IP地址：端口。

broadcastAddress（返回广播地址）

```c++
IPAddress::ptr IPv4Address::broadcastAddress(uint32_t prefix_len) {
    if (prefix_len > 32) {
        return nullptr;
    }

    sockaddr_in baddr(m_addr);
    baddr.sin_addr.s_addr |= byteswapOnLittleEndian(
        CreateMask<uint32_t>(prefix_len)); // 主机号全为1
    return IPv4Address::ptr(new IPv4Address(baddr));
}
```

networkAddress（返回网络地址）

```c++
IPAddress::ptr IPv4Address::networkAddress(uint32_t prefix_len) {
    if (prefix_len > 32) {
        return nullptr;
    }

    sockaddr_in baddr(m_addr);
    baddr.sin_addr.s_addr &= byteswapOnLittleEndian(
        ~CreateMask<uint32_t>(prefix_len));// 主机号全为0
    return IPv4Address::ptr(new IPv4Address(baddr));
}
```

subnetMask（返回子网掩码地址）

```c++
IPAddress::ptr IPv4Address::subnetMask(uint32_t prefix_len) {
    sockaddr_in subnet;
    memset(&subnet, 0, sizeof(subnet));
    subnet.sin_family      = AF_INET;
    subnet.sin_addr.s_addr = ~byteswapOnLittleEndian(CreateMask<uint32_t>(prefix_len));// 根据前缀长度获得子网掩码
    return IPv4Address::ptr(new IPv4Address(subnet));
}
```

getPort（返回端口号）

```c++
uint32_t IPv4Address::getPort() const {
    return byteswapOnLittleEndian(m_addr.sin_port);
}
```

setPort（设置端口号）

```c++
void IPv4Address::setPort(uint16_t v) {
    m_addr.sin_port = byteswapOnLittleEndian(v);
}
```

## *class* IPv6Address

mumber（成员变量）：`sockaddr_in6 m_addr;`

Create（通过IPv6地址字符串构造IPv6Address）

```c++
IPv6Address::ptr IPv6Address::Create(const char *address, uint16_t port) {
    IPv6Address::ptr rt(new IPv6Address);
    rt->m_addr.sin6_port = byteswapOnLittleEndian(port);
    int result           = inet_pton(AF_INET6, address, &rt->m_addr.sin6_addr);
    if (result <= 0) {
        SYLAR_LOG_DEBUG(g_logger) << "IPv6Address::Create(" << address << ", "<< port << ") rt=" << result << " errno=" << errno << " errstr=" << strerror(errno);
        return nullptr;
    }
    return rt;
}
```

IPv6Address（构造函数）

```c++
IPv6Address::IPv6Address() {
    memset(&m_addr, 0, sizeof(m_addr));
    m_addr.sin6_family = AF_INET6;
}

IPv6Address::IPv6Address(const sockaddr_in6 &address) {
    m_addr = address;
}

IPv6Address::IPv6Address(const uint8_t address[16], uint16_t port) {
    memset(&m_addr, 0, sizeof(m_addr));
    m_addr.sin6_family = AF_INET6;
    m_addr.sin6_port   = byteswapOnLittleEndian(port);
    memcpy(&m_addr.sin6_addr.s6_addr, address, 16);
}
```

getAddr（返回sockaddr指针）

```c++
sockaddr *IPv6Address::getAddr() {
    return (sockaddr *)&m_addr;
}

const sockaddr *IPv6Address::getAddr() const {
    return (sockaddr *)&m_addr;
}
```

getAddrLen（返回sockaddr的长度）

```c++
socklen_t IPv6Address::getAddrLen() const {
    return sizeof(m_addr);
}
```

insert（可读性输出地址）

```c++
std::ostream &IPv6Address::insert(std::ostream &os) const {
    os << "[";
    uint16_t *addr  = (uint16_t *)m_addr.sin6_addr.s6_addr;
    bool used_zeros = false;
    for (size_t i = 0; i < 8; ++i) {
        if (addr[i] == 0 && !used_zeros) {// 将连续0块省略
            continue;
        }
        if (i && addr[i - 1] == 0 && !used_zeros) {// 上一块为0，多输出个:
            os << ":";
            used_zeros = true;// 省略过了，后面不能再省略连续0块了
        }
        if (i) {
            os << ":";// 每个块后都要输出:
        }
        os << std::hex << (int)byteswapOnLittleEndian(addr[i]) << std::dec;// 按十六进制输出
    }
    // 若最后一块为0省略
    if (!used_zeros && addr[7] == 0) {
        os << "::";
    }

    os << "]:" << byteswapOnLittleEndian(m_addr.sin6_port);
    return os;
}
```

IPv6格式的标准输出，需要注意的是，IPv6是可以简写的，所以这里很多行代码都是在处理这个问题。

broadcastAddress（返回广播地址）

```c++
IPAddress::ptr IPv6Address::broadcastAddress(uint32_t prefix_len) {
    sockaddr_in6 baddr(m_addr);
    baddr.sin6_addr.s6_addr[prefix_len / 8] |=
        CreateMask<uint8_t>(prefix_len % 8);//找到前缀长度结尾在第几个字节，在该字节在哪个位置。将该字节前剩余位置全部置为1
    for (int i = prefix_len / 8 + 1; i < 16; ++i) {// 将后面其余字节置为1
        baddr.sin6_addr.s6_addr[i] = 0xff;
    }
    return IPv6Address::ptr(new IPv6Address(baddr));
}
```

networkAddress（返回网络地址）

```c++
IPAddress::ptr IPv6Address::networkAddress(uint32_t prefix_len) {
    sockaddr_in6 baddr(m_addr);
    baddr.sin6_addr.s6_addr[prefix_len / 8] &=
        CreateMask<uint8_t>(prefix_len % 8);//找到前缀长度结尾在第几个字节，在该字节在哪个位置。将该字节前剩余位置全部置为0
    for (int i = prefix_len / 8 + 1; i < 16; ++i) {// 将后面其余字节置为0
        baddr.sin6_addr.s6_addr[i] = 0x00;
    }
    return IPv6Address::ptr(new IPv6Address(baddr));
}
```

subnetMask（返回子网掩码地址）

```c++
IPAddress::ptr IPv6Address::subnetMask(uint32_t prefix_len) {
    sockaddr_in6 subnet;
    memset(&subnet, 0, sizeof(subnet));
    subnet.sin6_family = AF_INET6;
    subnet.sin6_addr.s6_addr[prefix_len / 8] =
        ~CreateMask<uint8_t>(prefix_len % 8);//找到前缀长度结尾在第几个字节，在该字节在哪个位置
    // 将前面的字节全部置为1
    for (uint32_t i = 0; i < prefix_len / 8; ++i) {
        subnet.sin6_addr.s6_addr[i] = 0xff;
    }
    return IPv6Address::ptr(new IPv6Address(subnet));
}
```

getPort（返回端口号）

```c++
uint32_t IPv6Address::getPort() const {
    return byteswapOnLittleEndian(m_addr.sin6_port);
}
```

setPort（设置端口号）

```c++
void IPv6Address::setPort(uint16_t v) {
    m_addr.sin6_port = byteswapOnLittleEndian(v);
}
```

## *class* UinxAddress

mumber（成员变量）

```c++
sockaddr_un m_addr;
socklen_t m_length;
```

UinxAddress（构造函数）

```c++
// Unix域套接字路径名的最大长度。-1是减去'\0'
static const size_t MAX_PATH_LEN = sizeof(((sockaddr_un *)0)->sun_path) - 1;

UnixAddress::UnixAddress() {// 默认构造
    memset(&m_addr, 0, sizeof(m_addr));
    m_addr.sun_family = AF_UNIX;
    m_length          = offsetof(sockaddr_un, sun_path) + MAX_PATH_LEN;// sun_path的偏移量+最大路径
}

UnixAddress::UnixAddress(const std::string &path) {// 通过路径
    memset(&m_addr, 0, sizeof(m_addr));
    m_addr.sun_family = AF_UNIX;
    m_length          = path.size() + 1;// 加上'\0'的长度

    if (!path.empty() && path[0] == '\0') {
        --m_length;
    }

    if (m_length > sizeof(m_addr.sun_path)) {
        throw std::logic_error("path too long");
    }
    memcpy(m_addr.sun_path, path.c_str(), m_length);// 将path放入结构体
    m_length += offsetof(sockaddr_un, sun_path);// 偏移量+路径长度
}
```

getAddrLen（设置sockaddr的长度）

```c++
void UnixAddress::setAddrLen(uint32_t v) {
    m_length = v;
}
```

getAddr（返回sockaddr指针）

```c++
sockaddr *UnixAddress::getAddr() {
    return (sockaddr *)&m_addr;
}
```

etAddrLen（返回sockaddr的长度）

```C++
socklen_t UnixAddress::getAddrLen() const {
    return m_length;
}
```

insert（可读性输出地址）

```c++
std::ostream &UnixAddress::insert(std::ostream &os) const {
    if (m_length > offsetof(sockaddr_un, sun_path) && m_addr.sun_path[0] == '\0') {
        return os << "\\0" << std::string(m_addr.sun_path + 1, m_length - offsetof(sockaddr_un, sun_path) - 1);
    }
    return os << m_addr.sun_path;
}
```

后面就是主函数了。

```c++
int main(int argc, char *argv[]) {
    sylar::EnvMgr::GetInstance()->init(argc, argv);
    sylar::Config::LoadFromConfDir(sylar::EnvMgr::GetInstance()->getConfigPath());

    // 获取本机所有网卡的IPv4地址和IPv6地址以及掩码长度
    test_ifaces(AF_INET);
    test_ifaces(AF_INET6);

    // 获取本机eth0网卡的IPv4地址和IPv6地址以及掩码长度，不过这个网卡好像没有mmm
    test_iface("eth0", AF_INET);
    test_iface("eth0", AF_INET6);

    // ip域名服务解析
    test_lookup("127.0.0.1");
    test_lookup("127.0.0.1:80");
    test_lookup("127.0.0.1:http");
    test_lookup("127.0.0.1:ftp");
    test_lookup("localhost");
    test_lookup("localhost:80");
    test_lookup("www.baidu.com");
    test_lookup("www.baidu.com:80");
    test_lookup("www.baidu.com:http");

    // IPv4地址类测试
    test_ipv4();

    // IPv6地址类测试
    test_ipv6();

    // Unix套接字地址类测试
    test_unix();

    return 0;
}
```

主要分为三类，第一类是对于网卡的处理，主要用到`getifaddrs`函数。第二类是ip域名服务解析，主要用到`getaddrinfo`函数。第三类就是创造一个ip地址，然后进行输出操作。

这部分还挺简单的其实，主要是对一些操作的验证。不涉及线程，甚至不涉及进程，就很简单。

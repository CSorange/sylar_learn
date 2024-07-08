# sylar学习14：序列化模块



### 概述

>序列化模块提供序列化与反序列化的功能，支持大端序和小端序，底层使用链表的形式存储数据，这样可以节省内存，管理内存碎片。该模块支持所有基本类型数据的写入和读取，可以选择固定字节长度或可变字节长度写入，使用可变字节长度时，使用`Zigzag`将有符号整型压缩后再进行`Varint`编码，这样能够节省大量内存空间。再写入数据时，将数据写入链表的最后一个节点中，若写不下时，创建新的节点继续写入数据。

### Varint & Zigzag

>`Varint`是一种使用一个或多个字节序列化整数的方法，会把整数编码为变长字节。对于32位整型数据经过`Varint`编码后需要10个字节。在实际场景中小数字的使用率远远多于大数字，因此通过`Varint`编码对于大部分场景都可以起到很好的压缩效果。
>
>`Zigzag`算法将有符号负整数转为正数，这样能够节省字节，因为负数的二进制位几乎全为1。

Varint处理编码和解码。

编码：使用**小端序**：**低位**字节存入数组**低地址**，**高位**字节存入数组**高地址**。

解码：数组**低地址**保存到数据的**低地址**，数组**高地址**保存到数据的**高地址**。

```c++
void EncodeVarint(uint32_t value) {//编码
    uint8_t temp[5];
    uint8_t i = 0;
    while(value > 127) {
        temp[i++] = (value & 0x7f) | 0x80;
        value >>= 7;
    }
    temp[i] = value;
    write(tmp, i);
}
void DecoderVarint() {//解码
    uint32_t result;
    for(uint i = 0; i < 32; i += 7) {
        uint8_t b = readFuint8();
        if(b < 0x80) {
            result |= (uint32_t)b << i;
            break;
        } else {
            result |= ((uint32_t)(b & 0x7f) << i);
        }
    }
}
```

Zigzag处理压缩和解压缩。压缩将负数x变为2\*(-x)-1，正数x变为2\*x。这个主要是为了消除负数，因为负数的二进制位几乎都为1。

```C++
void EncodeZigzag32(const int32_t value) {
    if(v < 0) {
        return ((uint32_t)(-v)) * 2 - 1;
    } else {
        return value * 2;
    }
}
void DecodeZigzag(const uint32_t& v) {
    return (v >> 1) ^ -(v & 1);
}
```



## *class* ByteArray

Node（链表结构）

```c++
struct Node {
        /**
         * @brief 构造指定大小的内存块
         * @param[in] s 内存块字节数
         */
        Node(size_t s);
        //无参构造函数
        Node();
        ~Node();
        /// 内存块地址指针
        char* ptr;
        /// 下一个内存块地址
        Node* next;
        /// 内存块大小
        size_t size;
    };
```

所有的数据都存在这个链表结构中，在堆区开辟内存存储在`ptr`中；`size`为一个节点大小，一般设置为一个页面大小4KB；`next`指向下一个节点。

成员变量：

```c++
/// 内存块的大小
size_t m_baseSize;
/// 当前操作位置
size_t m_position;
/// 当前的总容量
size_t m_capacity;
/// 当前数据的大小
size_t m_size;
/// 字节序,默认大端
int8_t m_endian;
/// 第一个内存块指针
Node* m_root;
/// 当前操作的内存块指针
Node* m_cur;
```

可以看到这里的总容量和大小是不同的，感觉有点像vector那种。

构造函数：

```c++
ByteArray::ByteArray(size_t base_size)
    :m_baseSize(base_size)
    ,m_position(0)
    ,m_capacity(base_size)
    ,m_size(0)
    ,m_endian(SYLAR_BIG_ENDIAN)
    ,m_root(new Node(base_size))
    ,m_cur(m_root) {
}
```

可以看到就是对成员变量的初始化。

析构函数：

```c++
ByteArray::~ByteArray() {
    Node* tmp = m_root;
    while(tmp) {
        m_cur = tmp;
        tmp = tmp->next;
        delete m_cur;
    }
}
```

相当于从第一个内存块开始删除，直到把所有的内存块都删掉为止。

ByteArray里面有很多为各种各样数据结构涉及的写入读出函数，这里就那两个进行列举。

```c++
void ByteArray::writeFint8  (int8_t value) {
    write(&value, sizeof(value));
}
int8_t   ByteArray::readFint8() {
    int8_t v;
    read(&v, sizeof(v));
    return v;
}
```

写入/读出长度int8_t类型的数据。

另外我们为什么要分开呢，一方面是写入的数据格式不同，另一方面是我们需要对不同的数据格式进行不同操作，例如如果大于8位，需要判断是大端还是小端这样。

clear（清空ByteArray）

```c++
void ByteArray::clear() {
    m_position = m_size = 0;
    m_capacity = m_baseSize;
    Node* tmp = m_root->next;
    while(tmp) {
        m_cur = tmp;
        tmp = tmp->next;
        delete m_cur;
    }
    m_cur = m_root;
    m_root->next = NULL;
}
```

和析构函数有点像，不过析构函数似乎只处理了关于内存块的部分。

write（写操作，改变position）

```c++
void ByteArray::write(const void* buf, size_t size) {
    if(size == 0) {
        return;
    }
    addCapacity(size);

    size_t npos = m_position % m_baseSize; //从第几块开始写入
    size_t ncap = m_cur->size - npos; //当前块还可以写多少
    size_t bpos = 0;

    while(size > 0) {
        if(ncap >= size) { //如果当前块就可以写入，就不需要考虑跨块的问题了
            memcpy(m_cur->ptr + npos, (const char*)buf + bpos, size);
            if(m_cur->size == (npos + size)) {
                m_cur = m_cur->next;
            }
            m_position += size;
            bpos += size;
            size = 0;
        } else { //当前块不能写入，就把当前块剩下的空间用完，再看下一块
            memcpy(m_cur->ptr + npos, (const char*)buf + bpos, ncap);
            m_position += ncap;
            bpos += ncap;
            size -= ncap;
            m_cur = m_cur->next;
            ncap = m_cur->size;
            npos = 0;
        }
    }

    if(m_position > m_size) {
        m_size = m_position;
    }
}
```

addCapacity（扩容）

```c++
void ByteArray::addCapacity(size_t size) {
    if(size == 0) {
        return;
    }
    size_t old_cap = getCapacity(); //还可以用的空间
    if(old_cap >= size) { //如果足够size这么大的数据写入，则不需要扩了
        return;
    }

    size = size - old_cap;
    size_t count = ceil(1.0 * size / m_baseSize);//看看需要添加几块
    Node* tmp = m_root;
    while(tmp->next) {
        tmp = tmp->next;
    }//到最后一块上去

    Node* first = NULL;
    for(size_t i = 0; i < count; ++i) {
        tmp->next = new Node(m_baseSize);
        if(first == NULL) {
            first = tmp->next;
        }
        tmp = tmp->next;
        m_capacity += m_baseSize;
    }

    if(old_cap == 0) {
        m_cur = first;
    }
}
```

read（读操作，改变position）

```c++
void ByteArray::read(void* buf, size_t size) {
    if(size > getReadSize()) {
        throw std::out_of_range("not enough len");
    }

    size_t npos = m_position % m_baseSize;
    size_t ncap = m_cur->size - npos;
    size_t bpos = 0;
    while(size > 0) {
        if(ncap >= size) {
            memcpy((char*)buf + bpos, m_cur->ptr + npos, size);
            if(m_cur->size == (npos + size)) {
                m_cur = m_cur->next;
            }
            m_position += size;
            bpos += size;
            size = 0;
        } else {
            memcpy((char*)buf + bpos, m_cur->ptr + npos, ncap);
            m_position += ncap;
            bpos += ncap;
            size -= ncap;
            m_cur = m_cur->next;
            ncap = m_cur->size;
            npos = 0;
        }
    }
}
```

write是写入，read是搞出来。

read（读操作，改变position）

```c++
void ByteArray::read(void* buf, size_t size, size_t position) const {
    if(size > (m_size - position)) {
        throw std::out_of_range("not enough len");
    }

    size_t npos = position % m_baseSize;
    size_t ncap = m_cur->size - npos;
    size_t bpos = 0;
    Node* cur = m_cur;
    while(size > 0) {
        if(ncap >= size) {
            memcpy((char*)buf + bpos, cur->ptr + npos, size);
            if(cur->size == (npos + size)) {
                cur = cur->next;
            }
            position += size;
            bpos += size;
            size = 0;
        } else {
            memcpy((char*)buf + bpos, cur->ptr + npos, ncap);
            position += ncap;
            bpos += ncap;
            size -= ncap;
            cur = cur->next;
            ncap = cur->size;
            npos = 0;
        }
    }
}
```

setPosition（设置当前位置）

```c++
void ByteArray::setPosition(size_t v) {
    if(v > m_capacity) {
        throw std::out_of_range("set_position out of range");
    }
    m_position = v;
    if(m_position > m_size) {
        m_size = m_position;
    }
    m_cur = m_root;
    while(v > m_cur->size) {
        v -= m_cur->size;
        m_cur = m_cur->next;
    }
    if(v == m_cur->size) {
        m_cur = m_cur->next;
    }
}
```

这个就不难，就不做展开了。

toString（转为string）将[m_position, m_size)转成std::string

```
std::string ByteArray::toString() const {
    std::string str;
    str.resize(getReadSize());
    if(str.empty()) {
        return str;
    }
    read(&str[0], str.size(), m_position);
    return str;
}
```

相似的还有toHexString。

之后是一组获取可读/可写的缓存，其中可读有两种情况，一种是从当前为止，另一种是从指定位置(这个就需要给定是从哪个位置开始的了)。

```c++
uint64_t ByteArray::getReadBuffers(std::vector<iovec>& buffers
                                ,uint64_t len, uint64_t position) const {
    len = len > getReadSize() ? getReadSize() : len;
    if(len == 0) {
        return 0;
    }

    uint64_t size = len;

    size_t npos = position % m_baseSize;
    size_t count = position / m_baseSize;
    Node* cur = m_root;
    while(count > 0) {
        cur = cur->next;
        --count;
    }//cur指向当前数据块

    size_t ncap = cur->size - npos;//position所在的块，剩下的空间。
    struct iovec iov;
    while(len > 0) {
        if(ncap >= len) {
            iov.iov_base = cur->ptr + npos;
            iov.iov_len = len;
            len = 0;
        } else {
            iov.iov_base = cur->ptr + npos;
            iov.iov_len = ncap;
            len -= ncap;
            cur = cur->next;
            ncap = cur->size;
            npos = 0;
        }
        buffers.push_back(iov);//最后得到的是一个数组，第一个和最后一个不满一个块，中间的都是满块
    }//用这个iov数组
    return size;
}

uint64_t ByteArray::getReadBuffers(std::vector<iovec>& buffers, uint64_t len) const {
    len = len > getReadSize() ? getReadSize() : len;
    if(len == 0) {
        return 0;
    }

    uint64_t size = len;

    size_t npos = m_position % m_baseSize;
    size_t ncap = m_cur->size - npos;
    struct iovec iov;
    Node* cur = m_cur;

    while(len > 0) {
        if(ncap >= len) {
            iov.iov_base = cur->ptr + npos;
            iov.iov_len = len;
            len = 0;
        } else {
            iov.iov_base = cur->ptr + npos;
            iov.iov_len = ncap;
            len -= ncap;
            cur = cur->next;
            ncap = cur->size;
            npos = 0;
        }
        buffers.push_back(iov);
    }
    return size;
}

uint64_t ByteArray::getWriteBuffers(std::vector<iovec>& buffers, uint64_t len) {
    if(len == 0) {
        return 0;
    }
    addCapacity(len);
    uint64_t size = len;

    size_t npos = m_position % m_baseSize;
    size_t ncap = m_cur->size - npos;
    struct iovec iov;
    Node* cur = m_cur;
    while(len > 0) {
        if(ncap >= len) {
            iov.iov_base = cur->ptr + npos;
            iov.iov_len = len;
            len = 0;
        } else {
            iov.iov_base = cur->ptr + npos;
            iov.iov_len = ncap;

            len -= ncap;
            cur = cur->next;
            ncap = cur->size;
            npos = 0;
        }
        buffers.push_back(iov);
    }
    return size;
}
```

这个其实就是模拟内存块的处理，我觉得这部分其实还是nginx更专业一点，自己可以之后再复习一下那个，然后用那个部分来讲。

测试主函数就不看了，感觉没有什么看的必要hh
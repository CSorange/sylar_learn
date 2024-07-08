# sylar学习17：Http模块

关于Http模块：

>主要封装HTTP请求（`class HttpRequest`）和HTTP响应报文（`class HttpResponse`）
>
>使用状态机解析报文格式，保存到请求和响应报文对象中。

请求报文格式：

```yaml
GET / HTTP/1.1  #请求行
Host: www.baidu.com   #主机地址
Connection: keep-alive   #表示TCP未断开
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64;x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.149 Safari/537.36   #产生请求的浏览器类型
Sec-Fetch-Dest: document
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: ......    #用户安全凭证
```

```bash
URI: http://www.sylar.top:80/page/xxx?id=10&v=20#fr
http  协议
www.sylar.top  host主机
80  post端口
/page/xxx 路径path
id=10&v=20 参数query
#fr fragment
```

应答报文格式

```xml
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Date: Mon, 05 Jun 2023 06:53:13 GMT
Link: <http://www.sylar.top/blog/index.php?rest_route=/>; rel="https://api.w.org/"
Server: nginx/1.12.2
Transfer-Encoding: chunked
X-Powered-By: PHP/7.0.33
Connection: close
Content-length: 45383

<!DOCTYPE html>
<html lang="zh-CN" class="no-js">
<head>
<meta charset="UTF-8">
```



HTTP方法枚举：

```c++
enum class HttpMethod {
#define XX(num, name, string) name = num,
    HTTP_METHOD_MAP(XX)
#undef XX
    INVALID_METHOD
};
```

HTTP状态枚举：

```
enum class HttpStatus {
#define XX(code, name, desc) name = code,
    HTTP_STATUS_MAP(XX)
#undef XX
};
```

之后就是关于类型转化的内容了。

```C++
HttpMethod StringToHttpMethod(const std::string& m);//将字符串方法名转成HTTP方法枚举
HttpMethod CharsToHttpMethod(const char* m);//将字符串指针转换成HTTP方法枚举
const char* HttpMethodToString(const HttpMethod& m);//将HTTP方法枚举转换成字符串
const char* HttpStatusToString(const HttpStatus& s);//将HTTP状态枚举转换成字符串
```

另外这里的比较是忽略大小写的

```c++
bool operator()(const std::string& lhs, const std::string& rhs) const;
bool CaseInsensitiveLess::operator()(const std::string &lhs, const std::string &rhs) const {
    return strcasecmp(lhs.c_str(), rhs.c_str()) < 0;
}
```

checkGetAs（获取Map中的key值,并转成对应类型,返回是否成功）

```c++
template<class MapType, class T>
bool checkGetAs(const MapType& m, const std::string& key, T& val, const T& def = T()) {
    auto it = m.find(key);
    if(it == m.end()) {
        val = def;
        return false;
    }
    try {
        val = boost::lexical_cast<T>(it->second);
        return true;
    } catch (...) {
        val = def;
    }
    return false;
}
```

getAs（获取Map中的key值,并转成对应类型）

```c++
template<class MapType, class T>
T getAs(const MapType& m, const std::string& key, const T& def = T()) {
    auto it = m.find(key);
    if(it == m.end()) {
        return def;
    }
    try {
        return boost::lexical_cast<T>(it->second);
    } catch (...) {
    }
    return def;
}
```

这两个函数其实很像，第一个返回值是是否找到了，找到的值存在val中。第二个返回值是找到的值。

## *class* HttpRequest

HTTP请求结构，主要提供方法赋值和取值。

成员函数：

```c++
/// HTTP方法
HttpMethod m_method;
/// HTTP版本
uint8_t m_version;
/// 是否自动关闭
bool m_close;
/// 是否为websocket
bool m_websocket;
/// 参数解析标志位，0:未解析，1:已解析url参数, 2:已解析http消息体中的参数，4:已解析cookies
uint8_t m_parserParamFlag;
/// 请求的完整url
std::string m_url;
/// 请求路径
std::string m_path;
/// 请求参数
std::string m_query;
/// 请求fragment
std::string m_fragment;
/// 请求消息体
std::string m_body;
/// 请求头部MAP
MapType m_headers;
/// 请求参数MAP
MapType m_params;
/// 请求Cookie MAP
MapType m_cookies;
```

构造函数：

```c++
HttpResponse(uint8_t version = 0x11, bool close = true);
HttpResponse::HttpResponse(uint8_t version, bool close)
    : m_status(HttpStatus::OK)
    , m_version(version)
    , m_close(close)
    , m_websocket(false) {
}
```

其中的成员函数大多是赋值或者返回值，就不一一列举了。

## *class* HttpResponse

HTTP响应结构体，主要提供方法赋值和取值。

```c++
/// 响应状态
HttpStatus m_status;
/// 版本
uint8_t m_version;
/// 是否自动关闭
bool m_close;
/// 是否为websocket
bool m_websocket;
/// 响应消息体
std::string m_body;
/// 响应原因
std::string m_reason;
/// 响应头部MAP
MapType m_headers;
/// cookies
std::vector<std::string> m_cookies;
```

构造函数：

```c++
HttpResponse(uint8_t version = 0x11, bool close = true);
HttpResponse::HttpResponse(uint8_t version, bool close)
    : m_status(HttpStatus::OK)
    , m_version(version)
    , m_close(close)
    , m_websocket(false) {
}
```

成员函数同上。

## *class* HttpRequestParser

HTTP请求解析类

成员变量

```c++
/// HTTP响应解析器
http_parser m_parser;
/// HTTP响应对象
HttpResponse::ptr m_data;
/// 错误码
int m_error;
/// 是否解析结束
bool m_finished;
/// 当前的HTTP头部field
std::string m_field;
```

构造函数：

```c++
HttpRequestParser::HttpRequestParser() {
    http_parser_init(&m_parser, HTTP_REQUEST);
    m_data.reset(new HttpRequest);
    m_parser.data = this;
    m_error       = 0;
    m_finished    = false;
}
void
http_parser_init (http_parser *parser, enum http_parser_type t)
{
  void *data = parser->data; /* preserve application data */
  memset(parser, 0, sizeof(*parser));
  parser->data = data;
  parser->type = t;
  parser->state = (t == HTTP_REQUEST ? s_start_req : (t == HTTP_RESPONSE ? s_start_res : s_start_req_or_res));
  parser->http_errno = HPE_OK;
}
```

execute执行函数

```c++
size_t HttpRequestParser::execute(char *data, size_t len) {
    size_t nparsed = http_parser_execute(&m_parser, &s_request_settings, data, len);
    if (m_parser.upgrade) {
        //处理新协议，暂时不处理
        SYLAR_LOG_DEBUG(g_logger) << "found upgrade, ignore";
        setError(HPE_UNKNOWN);
    } else if (m_parser.http_errno != 0) {
        SYLAR_LOG_DEBUG(g_logger) << "parse request fail: " << http_errno_name(HTTP_PARSER_ERRNO(&m_parser));
        setError((int8_t)m_parser.http_errno);
    } else {
        if (nparsed < len) {
            memmove(data, data + nparsed, (len - nparsed));// 解析完将剩余数据移动到起始地址
        }
    }
    return nparsed;
}
```

http_parser_execute看上去是真正的状态机处理函数，让我想起了编译原理。

其他的都是返回值或者赋值函数，就也不看了。

## *class* HttpResponseParser

Http响应解析结构体

成员变量

```
/// HTTP响应解析器
http_parser m_parser;
/// HTTP响应对象
HttpResponse::ptr m_data;
/// 错误码
int m_error;
/// 是否解析结束
bool m_finished;
/// 当前的HTTP头部field
std::string m_field;
```

emm似乎和上面的一样。

构造函数：

```c++
HttpRequestParser::HttpRequestParser() {
    http_parser_init(&m_parser, HTTP_REQUEST);
    m_data.reset(new HttpRequest);
    m_parser.data = this;
    m_error       = 0;
    m_finished    = false;
}
```

execute执行函数

```c++
size_t HttpRequestParser::execute(char *data, size_t len) {
    size_t nparsed = http_parser_execute(&m_parser, &s_request_settings, data, len);
    if (m_parser.upgrade) {
        //处理新协议，暂时不处理
        SYLAR_LOG_DEBUG(g_logger) << "found upgrade, ignore";
        setError(HPE_UNKNOWN);
    } else if (m_parser.http_errno != 0) {
        SYLAR_LOG_DEBUG(g_logger) << "parse request fail: " << http_errno_name(HTTP_PARSER_ERRNO(&m_parser));
        setError((int8_t)m_parser.http_errno);
    } else {
        if (nparsed < len) {
            memmove(data, data + nparsed, (len - nparsed));// 解析完将剩余数据移动到起始地址
        }
    }
    return nparsed;
}
```

可以看到http请求和http响应还是很相似的，其中主要的函数http_parser_init和http_parser_execute都是用的同一个。

主函数其实很简单：

```c++
test_http_request();
test_http_response();
void test_http_request() {
    sylar::http::HttpRequest req;
    req.setMethod(sylar::http::HttpMethod::GET);
    req.setVersion(0x11);
    req.setPath("/search");
    req.setQuery("q=url+%E5%8F%82%E6%95%B0%E6%9E%84%E9%80%A0&oq=url+%E5%8F%82%E6%95%B0%E6%9E%84%E9%80%A0+&aqs=chrome..69i57.8307j0j7&sourceid=chrome&ie=UTF-8");
    req.setHeader("Accept", "text/plain");
    req.setHeader("Content-Type", "application/x-www-form-urlencoded");
    req.setHeader("Cookie", "yummy_cookie=choco; tasty_cookie=strawberry");
    req.setBody("title=test&sub%5B1%5D=1&sub%5B2%5D=2"); // title=test&sub[1]=1&sub[2]=2

    req.dump(std::cout);

    std::cout << std::endl;
    std::cout << req.getParam("q") << std::endl;
    std::cout << req.getParam("title") << std::endl;
    std::cout << req.getParam("sub[1]") << std::endl;
    std::cout << req.getParam("sub[2]") << std::endl;
    std::cout << req.getHeader("Accept") << std::endl;
    std::cout << req.getCookie("yummy_cookie") << std::endl;
    std::cout << req.getCookie("tasty_cookie") << std::endl;
    std::cout << std::endl;
}

void test_http_response() {
    sylar::http::HttpResponse rsp;
    rsp.setStatus(sylar::http::HttpStatus::OK);
    rsp.setHeader("Content-Type", "text/html");
    rsp.setBody("<!DOCTYPE html>"
                "<html>"
                "<head>"
                "<title>hello world</title>"
                "</head>"
                "<body><p>hello world</p></body>"
                "</html>");
    rsp.setCookie("kookie1", "value1", 0, "/");
    rsp.setCookie("kookie2", "value2", 0, "/");
    rsp.dump(std::cout);
}
```

就是在赋值，然后输出到流中。

很简单，就不再过多去阐述了。
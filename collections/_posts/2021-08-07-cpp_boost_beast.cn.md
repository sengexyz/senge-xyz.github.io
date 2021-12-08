---
lang: cn
layout: post
title: "Boost::Beast Websocket库"
date: 2021-08-07
author: "[码农森哥](https://twitter.com/senge26430360)"
---


# Beast要点解析
## 概述
Beast是Boost中关于http(s)/websocket（s)的库，首发于boost 1.66（2016年），是比较新的库，它主要包含了http、websocket协议的解析（反序列化）和封装（序列化）以及关于网络的操作，它以asio为基础，但似乎又想隔离Asio。本文不是关于beast的全面描述，只涉及一些要点，主要资料来源于beast官方文档和实现代码。本文对应boost 的版本为1.73。


## 一、Beast中的Buffer
### 1.概念
 软件程序中，缓存区是被广泛应用的概念。Beast中的缓存区大多来自于Asio，DynamicBuffer是指由大小可变的缓存区域，根据不同内存方式，有三种形态：

* 一块连续内存，大小可变
* 多块内存，每块大小相同，可添加块
* 多块内存，每块大小不同，可添加块

BufferSequence其实是一个容器，将若干内存类包装起来（一般用双向链表），本身只提供遍历操作就可以了，它又分为可写的MutableBufferSequence和只读的ConstBufferSequence。MutableBufferSequence用于接收过程（即从网络接收到字节流，写入缓存），ConstBufferSequence可用于发送过程（即读出缓存区中已生成的字节流，发送到网络）。

Asio实现了两个“轻量级”的内存区域定义const_buffer和mutable_buffer，其实就是对struct {const void *, size_t }和struct {void *, size_t }这样结构的封装，内存的分配与释放由使用者负责，const_buffer和mutable_buffer只是引用而已，个人猜想实际的应用场景太多样了，如果将更多的功能添加到buffer上，来构造一个“重量级”的缓存管理，会影响性能，这与基础库的宗旨是不符的，因此Asio中只有一个streambuf类。

作为偏应用级别的Beast，其中有不少关于buffer的类，它进行了较大扩展，但意味着牺牲了性能和灵活性。个人感觉有点过头了，实际应用有各种要求，一致化的读写方式，显得比较繁琐，还是留给应用比较好。

以下是beast涉及缓存的类和辅助类。

### 2.flat_buffer

flat_buffer的定义如下：
```c++
using flat_buffer = basic_flat_buffer< std::allocator< char > >;
```

就是basic_flat_buffer的一个特化。

flat_buffer就相当于一个装内容的容器，外部写入的内容，对容器来说，就是读入的内容，外部写入区域可mutable，确认（commit）之后，就变成读入区的末尾，并且是只读的（const），在外部消费（consume）之后，清理读入区。
在flat_buffer内部实现时，采用了begin_,in_,out_,last_,end_五个指针来管理内存分配和各区域的大小，如下图所示。


要注意的是，单纯设置max_size不会引起内存的重新分配，只有在确实需要时才会，因此，Prepre, shrink_to_fit可能会重新分配内存。每次commit,consume,clear之后，会有相关指针的移动，因此建议从flat_buffer得到的读入或写入区域的指针，应即得即用，不要缓存，除非确认读入/写入区不会发生变化。

以下是一个flat_buffer的示例。

```c++
boost::beast::flat_buffer buffer(100);
buffer.max_size(200);
auto buf=buffer.prepare(10);       // buffer.prepare(201)会引发异常
char* const p = static_cast<char* const>(buf.data());
memcpy(p, "123456", 6);
buffer.commit(6);
auto cbuf = buffer.cdata();
char temp[10];
memcpy(temp, cbuf.data(), buffer.size());
```

上例中，先设定一个100字节的buffer，然后调整大小为200（此时并不重新分配，但会在后续操作中如此），如果申请(prepare)201大小的区域，则会产生异常。申请10字节大小写入区域，然后写入6个字符，最后查看读入区域，输出上述6个字符。输出如下：

### 3.multi_buffer
multi_buffer的定义如下：

```c++
using multi_buffer = basic_multi_buffer< std::allocator< char > >;
```

就是basic_multi_buffer的一个特化。

multi_buffer从功能上讲与flat_buffer是一样的，区别在于内存管理模式。flat_buffer的内存是一块连续区域（大小在运行时按需而变），multi_buffer的内存是一系列块（每块大小可以不相同），multi_buffer采用boost:: Intrusive库中的双向链表来管理，从整体上看，multi_buffer内存变得不连续了，操作上就麻烦多了，好处是不会发生内存大小调整了。

以下是一个multi_buffer的示例。

```c++
boost::beast::multi_buffer mbuffer(10);
auto buf1 = mbuffer.prepare(6);
for (auto it : buf1)
{
printf("writable:[%u]\n", it.size());
	memset(it.data(),'a', it.size());
}
mbuffer.commit(6);
mbuffer.max_size(30);
auto buf2 = mbuffer.prepare(7);
for (auto it : buf2)
{
	printf("writable:[%u]\n", it.size());
	memset(it.data(),'A', it.size());
}
mbuffer.commit(7);
auto bufs = mbuffer.data();
size_t aa=bufs.buffer_bytes();
printf("totalsize:%u\n",aa);
char temp[100];
for(auto it : bufs)  
{
	memcpy(temp, it.data(), it.size());
	for (int i = 0; i < it.size(); ++i)  printf("[%c]", temp[i]); printf("\n");
}
```

上例中，先设定一个10字节的buffer，此时会有一个10字节的内存块，写入（提交）6字节，然后再将buffer增大到30，则会有两个内存块（大小分别为10和（30-10），此时并不分配，但会变得如此），第二次写入（提交）7字节，其中4个字节写入第一块，3个字节写入第二块，最后将写入的内容打印出来，运行输出如下：


### 4.flat_static_buffer
flat_static_buffer的定义如下：

```c++
template< std::size_t N>
class flat_static_buffer : public flat_static_buffer_base
```

flat_static_buffer可认为是max_size在编译时就已确定的flat_buffer，因此内存大小固定，不会发生重新分配。基类flat_static_buffer_base引用外部分配的内存，定义了一系列操作，flat_static_buffer则分配了一段内存，其操作方法与基类基本一致（多了base()和operator=）。

以下是一个使用示例，与上面的示例基本一样。

```c++
boost::beast::flat_static_buffer<10> buf;
 
boost::asio::mutable_buffer  aa = buf.prepare(3);
memcpy(aa.data(), "aa\0", aa.size());
buf.commit(aa.size());
printf("capcity=%d read=%d\n", buf.capacity(),buf.size());
boost::asio::const_buffer  bb = buf.data();
printf("%s[%d]\n", static_cast<const char *>(bb.data()), bb.size());
buf.consume(aa.size());
printf("capcity=%d read=%d\n", buf.capacity(), buf.size());
```

### 5.static_buffer
static_buffer的定义如下：

```c++
template< std::size_t N>
class static_buffer : public static_buffer_base
```

static_buffer可认为是max_size在编译时就已确定，因此内存大小固定，不会发生重新分配。基类static_buffer_base引用外部分配的内存，定义了一系列操作，static_buffer则分配了一段内存，其操作方法与基类基本一致（多了base()和operator=）。
static_buffer与flat_static_buffer的不同在于它的内存结构是环状的（首尾逻辑相连），在某一时刻，读/写区域可能有两块，如下图所示，写区域有2块，读区域有1块或者写区域有1块，读区域有2块，因此相应的prepare/cdata所返回就是一对buffer。

static_buffer的用法与flat_buffer类似，以下是一个示例。

```c++
boost::beast::static_buffer<10> buf;
 
//以下为简洁可用auto声明，这里为清晰完整写出类型
boost::beast::detail::buffers_pair<true>  aa = buf.prepare(3);
printf("total=%d size1=%d  size2=%d\n", buf.capacity(),aa.begin()->size(), aa.end()->size());
 
memcpy(aa.begin()->data(), "aa\0", aa.begin()->size());
buf.commit(aa.begin()->size());
 
boost::beast::detail::buffers_pair<false>  bb = buf.cdata();
printf("%s[%d]  %s[%d]\n", (bb.begin()->size()>0)?(static_cast<const char*>(bb.begin()->data())):"", bb.begin()->size(),
		(bb.end()->size() > 0) ? (static_cast<const char*>(bb.end()->data())) : "", bb.end()->size());
buf.consume(bb.begin()->size()+ bb.end()->size());
printf("total=%d read=%d\n", buf.capacity(), buf.size());
```


### 6.buffers_adaptor
buffers_adaptor是将满足MutableBufferSequence要求的类包装成满足DynamicBuffer要求的类，内存的维护依然由原类负责。例如下面是将multi_buffer包装成一个类似flat_buffer的类，上面的static_buffer, flat_static_buffer也是满足DynamicBuffer要求的。

```c++
boost::beast::multi_buffer mbuffer(10);
boost::beast::buffers_adaptor<boost::beast::multi_buffer>  buf(mbuffer);
```

需要注意的是，此时，prepare/cdata返回的是多个块，需要遍历。

...

## 二、Beast中的string_view
Beast中的字符串操作，大量采用string_view形式，以代替const char *，由于std::string_view是C++17才提供的功能，因此beast中有一个宏BOOST_BEAST_USE_STD_STRING_VIEW来决定是采用std::string_view还是boost::string_view。

string_view其实就是以前Pascal风格的string（与之对应的就是C风格的字符串），此时字符串末尾没有’\0’，但增加一个字段表明它的长度，由此带来的副作用是以\0结尾为特征的字符串操作，例如输出、查找等，都不能再用了。

为兼容考虑，最初原始的字符串依然保留\0结尾，string_view仅是引用原始的字符串而已。其实std::string_view应该是作为std::string的“伴侣”而存在的（作为const char *的伴侣也可以）。当然，如果嫌麻烦，在自己代码中，干脆不用就好了（笔者就是这么做的）。

### 三、Beast中的http request和response
 在beast文档中，专门解释了是如何构思http request和response类的。Request和response的定义如下：

```c++
template<bool isRequest, class Fields, class Body>
class message:public header<isRequest, Fields>,boost::empty_value<typename Body::value_type>
```

以上的定义与文档中稍有不同，来自于源代码，文档中说message是一个容器，包含header与body，实际上message是继承于header与body，文档中的描述大致是为了更好理解，功能应该是一样的。当isRequest是true时就是request，是false时就是response。

Header的定义也是如此，文档中header有isRequest参数模板，但源码中被偏特化为两个类了，如下：

```c++
//request header
template<class Fields>
class header<true, Fields> : public Fields
//response header
template<class Fields>
class header<false, Fields> : public Fields
```

一般情况下，request和response基本上是没有关系的，是否将二者定义为同一个模板类，可能各人有各人的想法。

各类之间的关系如下图所示。出现模板的概念后，类之间的关系不好表达了，原来静态的关系变成了动态（模板实例化后其实是一个新类），因此下图仅是一个示意。

fields（basic_fields）是各种key-value的集合，该类内部有一个boost::intrusive::multiset和boost::intrusive::list，field被定义为enum class，集合保存的元素其实是（element+content），它是一段内存块，主体是element+name+value，其中element中包含了field，代码示意如下：

```c++
    auto const p = alloc_traits::allocate(a,
        (sizeof(element) + (name长)+ (value长) + 2 + sizeof(align_type) - 1) /
            sizeof(align_type));
*(::new(p) element(name, sname, value));
```

fields类提供了element的管理。

File_body等提供了对各种body类型的封装，基本上它包括三个内容

* reader类：内部类，需要满足BodyReader要求，将外部内容放入本类中（可能包含一些处理）
* writer类：内部类，将内容serialize后返回给外部类
* size：内容大小

beast这种构造request/response的方法，要求在最开始的时候就知道它的类型，然后采用已定义好的“模板”来装载它，如果这种流式信息的结构在开始时不能得到或不易得到（例如层级结构，还没收到完整的信息），怎么会有确定的“模板”？

beast作者将类型作为模板作为类构造的输入，可能与作者过于强调“模板”思路有关(beast自带例子中，很多是模板函数)，并没有一个通用的request和response基类，有时候会显得不自然，例如，当请求一个文件时，就会产生3种response：正常情况下的response<file body>；不存在文件下的response<empty body>(此时不定制错误信息)；异常情况下的response<string body>(此时定制错误信息)。我们通常的思路似乎是构造一个通用response（或基类），然后装载不同类（file/system error/string）的serialization后的结果。

以下是beast文档中提供的一个方案的例子，思路是首先构造empty_body的request，读取header，然后根据content_type的内容再将request修改（move）为string_body或dynamic_body类型。

```c++
request_parser<empty_body> req0;
read_header(stream, buffer, req0);
switch(req0.get().method())
{
    case verb::post:
    {
        if( req0.get()[field::content_type] != "application/x-www-form-urlencoded" &&
            req0.get()[field::content_type] != "multipart/form-data")
            goto do_dynamic_body;
        request_parser<string_body> req{std::move(req0)};
        read(stream, buffer, req);
        handler(req.release());
        break;
    }
    do_dynamic_body:
    default:
    {
       request_parser<dynamic_body> req{std::move(req0)};
        read(stream, buffer, req);
        handler(req.release());
        break;
    }
}
```

### chunked request/response

chunked body不同于一个完整的body，它由若干如下结构的块构成：

XX:extension\r\n

content\r\n

其中XX（十六进制）表示下面content的长度，extension可选，是形如k1[=v1];k2[=v2]这样的key=value对(value可选)，content是本块的内容。

Beast提供了以下类：

chunk_body：表示一个块，由chunk_size，chunk_extensions构成的const_buffer, chunk_crlf，内容构成的ConstBufferSequence和chunk_crlf构成。
chunk_crlf：就是\r\n
chunk_extensions：一系列的key=value
chunk_header：表示一个块内容之前的部分，即chunk_size，chunk_extensions构成的const_buffer和 chunk_crlf。
chunk_last：最后一块，可以添加trailer。
2个函数make_chunk,make_chunk_last用于生成chunk_body和chunk_last。

使用的流程大致是（以reponse为例）：

```c++
response<empty_body> res{status::ok, 11};           //构造response
res.set(…);            //设置field
res.chunked(true);       //设置chunked模式
 
response_serializer<empty_body> sr{res};         //写header
write_header(sock, sr);
 
chunk_extensions ext;         //构造extension
ext.insert(…);
 
// 写chunk块, chunk块的内容已被写入const_buffer（可参见buffer的用法）
net::write(sock, make_chunk(const_buffer,ext));
…
net::write(sock, make_chunk_last());     //写入最后块
```

有关chunk的结构设计得比较麻烦，但通过make_chunk,make_chunk_last，使用起来并不麻烦。


## 四、Beast中的network
由于http、websocket仅涉及tcp，因此在beast范围内，也仅涉及tcp协议，Beast的网络操作基于Asio，但Beast的一个目标似乎是做一个完整的系统（猜测），因此beast将涉及到的网络数据操作都“重写”的一遍（一些是简单包装，一些是扩展），例如Asio空间被重新命名为net,对std:bind也扩展为bind_handler和bind_front_handler。

beast设计了两套stream，一是用于http的tcp_stream，另一个是用于websocket的stream，它们之间没有直接关系。
tcp_stream的相关定义如下：

```c++
template< class Protocol, class Executor = net::executor,class RatePolicy = unlimited_rate_policy>
class basic_stream
using tcp_stream = basic_stream< net::ip::tcp, net::executor, unlimited_rate_policy >;
```

从实现上看，beast并没有利用asio中的streambuf，而是采用其中的概念，与buffer类结合，重新写了一大套接收/发送操作，与Asio中类似。

在实现websocket时，beast作者试图体现网络分层的概念，采用get_lowest_layer()，get_next_layer()来获得更下层的实现类，websocket中的stream定义如下：

```c++
template<class NextLayer, bool deflateSupported>
class stream
```

其中deflateSupported是websocket协议扩展，表示是否支持报文压缩，如果定义一个ws流:

websocket::stream< boost::beast::tcp_stream > ws(ioc);

则上面那个ws的next layer就是boost::beast::tcp_stream。

同样，beast重新写了一大套接收/发送操作。

实际上，在basic_stream的实现代码中也可看到类似layer的身影，但并没有在文档中出现，猜测作者认为还不成熟，暂且在websocket中实验，估计在以后的版本中，网络分层的概念的会被强化。


## 五、Beast中的bind
bind_handler没有当内置handler是成员函数时的情况，其他与std:bind似乎没有什么区别。

bind_front_handler功能基本同std:bind，所不同的是handler参数的写法，这里不能使用std::placeholders，handler函数所有原来需要std::placeholders的参数只能设计在后面（还是看下面的例子吧，注意两种方式写法的不同）。bind_front_handler显得简洁一些。

```c++
class AA
{
public:
	void handler(int i, float f)
	{
		printf("i=%d f=%f\n", i, f);
	}
	void trigger(std::function<void(float)> h)
	{
		h(1.1);
	}
	void run()
	{
		trigger(boost::beast::bind_front_handler(&AA::handler, this, 2));
		trigger(std::bind(&AA::handler, this, 2, std::placeholders::_1));
	}
};
```

## 六、Beast中的timeout
Asio中基本的socket并没有附带超时操作(iostream有)，但实际的网络操作都不可避免的要设置超时，因此beast在其网络数据流基类basic_stream中提供了超时设置功能：expires_after，expires_at以及expires_never，对于客户端，显得十分方便，但对于服务器，如果有很多连接，设置如此多的定时器，必然会增加机器的负担，使用时应该权衡。

在beast实现代码中，设置了读和写两个定时器，其本意大概是可设置读超时和写超时，但却没有分开设置的接口，现在使用的方式是在操作之前设置，如在读操作前设置一次超时，对读操作有效，在写操作之前设置一次超时，对写操作有效。

## 七、Websocket
Websocket API比较少，感觉作者的重心不在此处。

Websocket通信是由http通信“upgrade”而来，因此Websocket通信的第一步依然是构造一个http request parser，parser检查到需要upgrade到websocket协议后，构造websocket流，它的定义如下：

```c++
websocket::stream< boost::beast::tcp_stream > ws(ioc);
```

然后，ws接受连接(accept)，通过ws接收和发送数据。

Websocket读写没有相应的request/response类，因此与一般的tcp流使用类似，自己写解析与应答。

Websocket采用set_option来进行设置，一个通常的设置是超时（作者建议采用这种方法，而不用更下层的方法，并且在构造之前需要disable下层设置的超时）。Websocke有三个超时：握手时超时，正常通信时超时，当达到正常通信时超时半数时是否发送ping信息而保持连接。以下是一个示例（先disable掉tcpstream的超时似乎有点不简洁）。

```c++
tcpstream.expires_never();
websocket::stream< boost::beast::tcp_stream > ws(std::move(tcpstream));
stream_base::timeout opt{
    std::chrono::seconds(30),   // handshake timeout
    stream_base::none(),      // disable idle timeout
    false                  //not set ping
};
ws.set_option(opt);
```

## 八、总结：
1.  Beast运用了boost的很多东西，很好地运用了template的特性，但使得学习成本高，看一个东西牵扯其他很多东西，对库的开发者来说，这不是问题，但对普通应用者来说，就是大问题。

2. 作为偏向协议解析的库，Beast涉及很多网络操作，反而显得与协议解析部分绑定的较紧，例如，

```c++
read(stream, buffer, request)；
```

这样的功能感觉上分两步更灵活：read stream into buffer和 parse buffer into request，其中第一步由Asio或其他别的网络库来完成（以目前的实现，如果不采用Asio，真不知如何）。Beast中提供的API，涉及网络及相关的操作不算少数，并且提出了next layer, decorator等概念，目标比较宏大，但却有点偏离协议解析这个最基本的目标。实际的应用中，离不开诸如url encode,querystring parsing等功能，做web服务器应用时，还需要url routing，这些实用的功能，beast反而没有提供，所以有时会迷惑，beast到底定位成什么库呢。
3.    TCP数据就是流式数据，无论request还是response都是char序列（流），如果细分，还可以是text流或binary流，接收和发送由网络库负责，接收到时，由request卸载；发送前由response装载，负荷是其他具体类型类（与协议对应）的deserialization /serialization，以上是通常的思路，beast将类型确定提前了，由类型类才能构造出request/response，这种思路是否更好，见仁见智吧，但使用beas写代码时，应该适应这种思路。
4.    Beast提供了协议解析的一些新思路，形式的优美与运行的高效之间如何平衡，作为库作者，一般是强调前者，作为项目的作者，则一般强调后者。希望beast能找到一个兼容的方案。
5.    Asio已发展了若干版本，从每次boost版本的更新文档中，都可以看到Asio每次不断的努力，beast定位比较宏大，感觉还有很长的路走。


___
---
***


# Boost Beast 1.69 随笔

在1.69.0中没有以下的类型

1. tcp_stream
2. boost_front_handler

## request和response body类型

1. 首先request和response都属于message，message主要有header和body
其中header主要就是http基本信息，如GET/POST 方法等

2. message本身是个template，主要有一下几个类型

* http::empty_body
顾名思义就是没有body的message，一般用于不含有pay_load GET方法的request
例子：
```c++
namespace http = boost::beast::http;
http::request<http::empty_body> req_;
req_.version(11);  // http1.1
req_.method(http::verb::get); // get method
req_.target("/"); // 一个URL host:port 后面的部分，如 http://127.0.0.1:80/abc/123 , 那么 target = /abc/123
req_.set(http::field::host, "127.0.0.1");
req_.set(http::field::user_agent,BOOST_BEAST_VERSION_STRING);
```

* http::string_body
```c++
namespace http = boost::beast::http;
http::string_body::value_type body;
body = "some message";
http::response<http::string_body> res{http::status::not_found, 11};
res.set(http::field::server, BOOST_BEAST_VERSION_STRING);
res.set(http::field::content_type, "text/html");
res.body() = std::move(body);
res.prepare_payload();
```

* http::file_body
```c++
namespace http = boost::beast::http;
http::file_body::value_type body;
beast::error_code ec;
body.open("path", beast::file_mode::scan, ec);
http::response<http::file_body> res{http::status::not_found, 11};
res.set(http::field::server, BOOST_BEAST_VERSION_STRING);
res.set(http::field::content_type, "text/html");
res.body() = std::move(body);
res.prepare_payload();
```

* http::buffer_body
* http::span_body<class T>
看源码都是连续内存buffer的body

### field
field主要是指http header中各种字段如content-type等
所有的field请参见beast源码中 include/boost/beast/http/field.hpp

* 设置方法
```c++
http::request<http::string_body> req;
req.set(http::field::host, host);
req.set(http::field::user_agent, BOOST_BEAST_VERSION_STRING);
req.set(http::field::content_type, "text/html");
req.set(http::field::accept_encoding, "gzip, deflate");
```

* 查找和获取
```c++
if (req.find(http::field::content_type) != req.end() )
{
  // find the field
  boost::string_view = req.at(http::field::content_type);
 // 或者使用[]操作符
}
```

* 插入
```c++
req.insert(http::field::content_type, "text/html");
```

* 删除
```c++
req.erase(http::field::content_type);
```

* 清空所有
```
req.clear();
```

___
---
***


# boost库websocket服务器

基于boost标准C++库，使用协程和beast实现子协议websocket服务器。作为初学者实现内容也比较简单，不做过多的解释，就直接上代码了。

```c++
//------------------------------------------------------------------------------
#include <boost/beast/core.hpp>
#include <boost/beast/http.hpp>
#include <boost/beast/version.hpp>
#include <boost/asio/ip/tcp.hpp>
#include <boost/asio/spawn.hpp>
#include <boost/config.hpp>
#include <algorithm>
#include <cstdlib>
#include <iostream>
#include <memory>
#include <string>
#include <thread>
#include <vector>
#include <boost/beast/websocket.hpp>
#include <boost/asio.hpp>
#include <boost/bind/bind.hpp>
#include <boost/date_time/posix_time/posix_time.hpp>
 
 
namespace beast = boost::beast;         // from <boost/beast.hpp>
namespace http = beast::http;           // from <boost/beast/http.hpp>
namespace net = boost::asio;            // from <boost/asio.hpp>
namespace websocket = beast::websocket; // from <boost/beast/websocket.hpp>
using tcp = boost::asio::ip::tcp;       // from <boost/asio/ip/tcp.hpp>
 
using namespace boost::asio;
using namespace boost::placeholders;
//------------------------------------------------------------------------------
// Report a failure
void fail(beast::error_code ec, char const* what)
{
    std::cerr << what << ": " << ec.message() << "\n";
}
 
template<class Body, class Allocator>
 
 
void do_session(
	net::io_context& ioc,
    websocket::stream<beast::tcp_stream>& ws,
	http::request<http::string_body> const& req,
    net::yield_context yield)
{
    beast::error_code ec;
	std::string protocol;
    deadline_timer timer(ioc);
	timer.expires_from_now(boost::posix_time::seconds(1));
	
	protocol.assign(req.target().begin(), req.target().end());
    if (protocol.empty()){
		return ;
	}	
    // Set suggested timeout settings for the websocket
    ws.set_option(websocket::stream_base::timeout::suggested(beast::role_type::server));
 
    // Set a decorator to change the Server of the handshake
	ws.set_option(websocket::stream_base::decorator(
		[protocol](websocket::response_type& res)
		{
			res.set(http::field::server, protocol);
		}));
 
    // Accept the websocket handshake
    ws.async_accept(req, yield[ec]);	
    if(ec) {
        return fail(ec, "accept");
	}
	if(0 == protocol.compare("/aa/lalala")) {
		for(;;)
		{
			// This buffer will hold the incoming message
			beast::flat_buffer buffer;
 
			ws.async_write(boost::asio::buffer("------- send lalala -------"), yield[ec]);
			if(ec == websocket::error::closed) {
				break;
			}
			if(ec){
				return fail(ec, "write"); 
			}
			timer.async_wait(yield[ec]);
		}		
	} else if(0 == protocol.compare("/aa/gagaga")) {
		for(;;)
		{
			// This buffer will hold the incoming message
			beast::flat_buffer buffer;
 
			ws.async_write(boost::asio::buffer("------- send gagaga-------"), yield[ec]);
			if(ec == websocket::error::closed) {
				break;
			}
			if(ec){
				return fail(ec, "write"); 
			}
			timer.async_wait(yield[ec]);
		}		
	} else if(0 == protocol.compare("/aa/hahaha")) {
		for(;;)
		{
			// This buffer will hold the incoming message
			beast::flat_buffer buffer;
 
			ws.async_write(boost::asio::buffer("------- send hahaha -------"), yield[ec]);
			if(ec == websocket::error::closed) {
				break;
			}
			if(ec){
				return fail(ec, "write"); 
			}
			timer.async_wait(yield[ec]);
		}
	} else {
		return ;
	}
}
 
 
// Accepts incoming connections and launches the sessions
void
do_listen(
    net::io_context& ioc,
    tcp::endpoint endpoint,
    net::yield_context yield)
{
    beast::error_code ec;
 
    // Open the acceptor
    tcp::acceptor acceptor(ioc);
    acceptor.open(endpoint.protocol(), ec);
    if(ec)
        return fail(ec, "open");
 
    // Allow address reuse
    acceptor.set_option(net::socket_base::reuse_address(true), ec);
    if(ec)
        return fail(ec, "set_option");
 
    // Bind to the server address
    acceptor.bind(endpoint, ec);
    if(ec)
        return fail(ec, "bind");
 
    // Start listening for connections
    acceptor.listen(net::socket_base::max_listen_connections, ec);
    if(ec)
        return fail(ec, "listen");
 
    for(;;)
    {
        tcp::socket socket(ioc);
        acceptor.async_accept(socket, yield[ec]);
        if(ec) {
            fail(ec, "accept");
        } else {
			beast::flat_buffer buffer;
			http::request<http::string_body> req;
			
			http::async_read(socket, buffer, req, yield[ec]);
			// 查看是否是WebSocket升级请求
			if(websocket::is_upgrade(req))
			{
					boost::asio::spawn(
						acceptor.get_executor(),
						std::bind(
							&do_session,
							std::ref(ioc),
							websocket::stream<beast::tcp_stream>(std::move(socket)),
							std::ref(req),
							std::placeholders::_1));
			}
		}
	}
}
 
int main(int argc, char* argv[])
{
    // Check command line arguments.
    if (argc != 4)
    {
        std::cerr <<
            "Usage: http-server-coro <address> <port> <doc_root> <threads>\n" <<
            "Example:\n" <<
            "    http-server-coro 0.0.0.0 8080 1\n";
        return EXIT_FAILURE;
    }
    auto const address = net::ip::make_address(argv[1]);
    auto const port = static_cast<unsigned short>(std::atoi(argv[2]));
    auto const threads = std::max<int>(1, std::atoi(argv[3]));
 
    // The io_context is required for all I/O
    net::io_context ioc{threads};
 
    // Spawn a listening port
    boost::asio::spawn(ioc,
        std::bind(
            &do_listen,
            std::ref(ioc),
            tcp::endpoint{address, port},
            std::placeholders::_1));
	
    // Run the I/O service on the requested number of threads
    std::vector<std::thread> v;
    v.reserve(threads - 1);
    for(auto i = threads - 1; i > 0; --i)
        v.emplace_back(
        [&ioc]
        {
            ioc.run();
        });
    ioc.run();
 
    return EXIT_SUCCESS;
}
```

___
---
***

# C++ 使用boost实现http客户端——同步、异步、协程
## 同步方式

```c++
#include <boost/beast/core.hpp>
#include <boost/beast/http.hpp>
#include <boost/beast/version.hpp>
#include <boost/asio/connect.hpp>
#include <boost/asio/ip/tcp.hpp>
#include <cstdlib>
#include <iostream>
#include <string>

using tcp = boost::asio::ip::tcp;       // from <boost/asio/ip/tcp.hpp>
namespace http = boost::beast::http;    // from <boost/beast/http.hpp>

										// Performs an HTTP GET and prints the response
int main(int argc, char** argv)
{
	try
	{
		auto const host = "www.baidu.com";		//要访问的主机名
		auto const port = "80";					//http服务端口
		auto const target = "/loading.html";	//要获取的文档
		int version = 11;		

		// The io_context is required for all I/O
		boost::asio::io_context ioc;

		// These objects perform our I/O
		tcp::resolver resolver{ ioc };
		tcp::socket socket{ ioc };

		// Look up the domain name
		auto const results = resolver.resolve(host, port);

		// Make the connection on the IP address we get from a lookup
		boost::asio::connect(socket, results.begin(), results.end());

		// Set up an HTTP GET request message
		http::request<http::string_body> req{ http::verb::get, target, version };
		req.set(http::field::host, host);
		req.set(http::field::user_agent, BOOST_BEAST_VERSION_STRING);

		// Send the HTTP request to the remote host
		http::write(socket, req);

		// This buffer is used for reading and must be persisted
		boost::beast::flat_buffer buffer;

		// Declare a container to hold the response
		http::response<http::dynamic_body> res;

		// Receive the HTTP response
		http::read(socket, buffer, res);

		
		// Write the message to standard out
		std::cout << res << std::endl;

		// Gracefully close the socket
		boost::system::error_code ec;
		socket.shutdown(tcp::socket::shutdown_both, ec);

		// not_connected happens sometimes
		// so don't bother reporting it.
		//
		if (ec && ec != boost::system::errc::not_connected)
			throw boost::system::system_error{ ec };

		// If we get here then the connection is closed gracefully
	}
	catch (std::exception const& e)
	{
		std::cerr << "Error: " << e.what() << std::endl;
		return EXIT_FAILURE;
	}
	return EXIT_SUCCESS;
}
```


## 异步方式
```c++
/
// Copyright (c) 2016-2017 Vinnie Falco (vinnie dot falco at gmail dot com)
//
// Distributed under the Boost Software License, Version 1.0. (See accompanying
// file LICENSE_1_0.txt or copy at http://www.boost.org/LICENSE_1_0.txt)
//
// Official repository: https://github.com/boostorg/beast
//

//------------------------------------------------------------------------------
//
// Example: HTTP client, asynchronous
//
//------------------------------------------------------------------------------

#include <boost/beast/core.hpp>
#include <boost/beast/http.hpp>
#include <boost/beast/version.hpp>
#include <boost/asio/connect.hpp>
#include <boost/asio/ip/tcp.hpp>
#include <cstdlib>
#include <functional>
#include <iostream>
#include <memory>
#include <string>


using tcp = boost::asio::ip::tcp;       // from <boost/asio/ip/tcp.hpp>
namespace http = boost::beast::http;    // from <boost/beast/http.hpp>

										//------------------------------------------------------------------------------

										// Report a failure
void
fail(boost::system::error_code ec, char const* what)
{
	std::cerr << what << ": " << ec.message() << "\n";
}

// Performs an HTTP GET and prints the response
class session : public std::enable_shared_from_this<session>
{
	tcp::resolver resolver_;
	tcp::socket socket_;
	boost::beast::flat_buffer buffer_; // (Must persist between reads)
	http::request<http::empty_body> req_;
	http::response<http::string_body> res_;

public:
	// Resolver and socket require an io_context
	explicit
		session(boost::asio::io_context& ioc)
		: resolver_(ioc)
		, socket_(ioc)
	{
	}

	// Start the asynchronous operation
	void
		run(
			char const* host,
			char const* port,
			char const* target,
			int version)
	{
		// Set up an HTTP GET request message
		req_.version(version);
		req_.method(http::verb::get);
		req_.target(target);
		req_.set(http::field::host, host);
		req_.set(http::field::user_agent, BOOST_BEAST_VERSION_STRING);

		// Look up the domain name
		resolver_.async_resolve(
			host,
			port,
			std::bind(
				&session::on_resolve,
				shared_from_this(),
				std::placeholders::_1,
				std::placeholders::_2));
	}

	void
		on_resolve(
			boost::system::error_code ec,
			tcp::resolver::results_type results)
	{
		if (ec)
			return fail(ec, "resolve");

		// Make the connection on the IP address we get from a lookup
		boost::asio::async_connect(
			socket_,
			results.begin(),
			results.end(),
			std::bind(
				&session::on_connect,
				shared_from_this(),
				std::placeholders::_1));
	}

	void
		on_connect(boost::system::error_code ec)
	{
		if (ec)
			return fail(ec, "connect");

		// Send the HTTP request to the remote host
		http::async_write(socket_, req_,
			std::bind(
				&session::on_write,
				shared_from_this(),
				std::placeholders::_1,
				std::placeholders::_2));
	}

	void
		on_write(
			boost::system::error_code ec,
			std::size_t bytes_transferred)
	{
		boost::ignore_unused(bytes_transferred);

		if (ec)
			return fail(ec, "write");

		// Receive the HTTP response
		http::async_read(socket_, buffer_, res_,
			std::bind(
				&session::on_read,
				shared_from_this(),
				std::placeholders::_1,
				std::placeholders::_2));
	}

	void
		on_read(
			boost::system::error_code ec,
			std::size_t bytes_transferred)
	{
		boost::ignore_unused(bytes_transferred);

		if (ec)
			return fail(ec, "read");

		// Write the message to standard out
		std::cout << res_ << std::endl;

		// Gracefully close the socket
		socket_.shutdown(tcp::socket::shutdown_both, ec);

		// not_connected happens sometimes so don't bother reporting it.
		if (ec && ec != boost::system::errc::not_connected)
			return fail(ec, "shutdown");

		// If we get here then the connection is closed gracefully
	}
};

//------------------------------------------------------------------------------

int main(int argc, char** argv)
{
#if 0
	// Check command line arguments.
	if (argc != 4 && argc != 5)
	{
		std::cerr <<
			"Usage: http-client-async <host> <port> <target> [<HTTP version: 1.0 or 1.1(default)>]\n" <<
			"Example:\n" <<
			"    http-client-async www.example.com 80 /\n" <<
			"    http-client-async www.example.com 80 / 1.0\n";
		return EXIT_FAILURE;
	}
#endif
	auto const host = "www.baidu.com";		//要访问的主机名
	auto const port = "80";					//http服务端口
	auto const target = "/loading.html";	//要获取的文档
	int version = 11;

	// The io_context is required for all I/O
	boost::asio::io_context ioc;

	// Launch the asynchronous operation
	std::make_shared<session>(ioc)->run(host, port, target, version);

	// Run the I/O service. The call will return when
	// the get operation is complete.
	ioc.run();

	return EXIT_SUCCESS;
}
```

## 协程方式
```c++
//
// Copyright (c) 2016-2017 Vinnie Falco (vinnie dot falco at gmail dot com)
//
// Distributed under the Boost Software License, Version 1.0. (See accompanying
// file LICENSE_1_0.txt or copy at http://www.boost.org/LICENSE_1_0.txt)
//
// Official repository: https://github.com/boostorg/beast
//

//------------------------------------------------------------------------------
//
// Example: HTTP client, coroutine
//
//------------------------------------------------------------------------------

#include <boost/beast/core.hpp>
#include <boost/beast/http.hpp>
#include <boost/beast/version.hpp>
#include <boost/asio/connect.hpp>
#include <boost/asio/spawn.hpp>
#include <boost/asio/ip/tcp.hpp>
#include <cstdlib>
#include <functional>
#include <iostream>
#include <string>

using tcp = boost::asio::ip::tcp;       // from <boost/asio/ip/tcp.hpp>
namespace http = boost::beast::http;    // from <boost/beast/http.hpp>

										//------------------------------------------------------------------------------

										// Report a failure
void
fail(boost::system::error_code ec, char const* what)
{
	std::cerr << what << ": " << ec.message() << "\n";
}

// Performs an HTTP GET and prints the response
void
do_session(
	std::string const& host,
	std::string const& port,
	std::string const& target,
	int version,
	boost::asio::io_context& ioc,
	boost::asio::yield_context yield)
{
	boost::system::error_code ec;

	// These objects perform our I/O
	tcp::resolver resolver{ ioc };
	tcp::socket socket{ ioc };

	// Look up the domain name
	auto const results = resolver.async_resolve(host, port, yield[ec]);
	if (ec)
		return fail(ec, "resolve");

	// Make the connection on the IP address we get from a lookup
	boost::asio::async_connect(socket, results.begin(), results.end(), yield[ec]);
	if (ec)
		return fail(ec, "connect");

	// Set up an HTTP GET request message
	http::request<http::string_body> req{ http::verb::get, target, version };
	req.set(http::field::host, host);
	req.set(http::field::user_agent, BOOST_BEAST_VERSION_STRING);

	// Send the HTTP request to the remote host
	http::async_write(socket, req, yield[ec]);
	if (ec)
		return fail(ec, "write");

	// This buffer is used for reading and must be persisted
	boost::beast::flat_buffer b;

	// Declare a container to hold the response
	http::response<http::dynamic_body> res;

	// Receive the HTTP response
	http::async_read(socket, b, res, yield[ec]);
	if (ec)
		return fail(ec, "read");

	// Write the message to standard out
	std::cout << res << std::endl;

	// Gracefully close the socket
	socket.shutdown(tcp::socket::shutdown_both, ec);

	// not_connected happens sometimes
	// so don't bother reporting it.
	//
	if (ec && ec != boost::system::errc::not_connected)
		return fail(ec, "shutdown");

	// If we get here then the connection is closed gracefully
}

//------------------------------------------------------------------------------

int main(int argc, char** argv)
{
#if 0
	// Check command line arguments.
	if (argc != 4 && argc != 5)
	{
		std::cerr <<
			"Usage: http-client-async <host> <port> <target> [<HTTP version: 1.0 or 1.1(default)>]\n" <<
			"Example:\n" <<
			"    http-client-async www.example.com 80 /\n" <<
			"    http-client-async www.example.com 80 / 1.0\n";
		return EXIT_FAILURE;
	}
#endif
	auto const host = "www.baidu.com";		//要访问的主机名
	auto const port = "80";					//http服务端口
	auto const target = "/loading.html";	//要获取的文档
	int version = 11;

	// The io_context is required for all I/O
	boost::asio::io_context ioc;

	// Launch the asynchronous operation
	boost::asio::spawn(ioc, std::bind(
		&do_session,
		std::string(host),
		std::string(port),
		std::string(target),
		version,
		std::ref(ioc),
		std::placeholders::_1));

	// Run the I/O service. The call will return when
	// the get operation is complete.
	ioc.run();

	return EXIT_SUCCESS;
}
```





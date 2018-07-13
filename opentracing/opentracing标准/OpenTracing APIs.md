# OpenTracing APIs
## OpenTracing的多语言支持
目前官方OpenTracing API支持的平台有：[Go](https://github.com/opentracing/opentracing-go)、[Python](https://github.com/opentracing/opentracing-python)、[JS](https://github.com/opentracing/opentracing-javascript)、[Objective-C](https://github.com/opentracing/opentracing-objc)、[Java](https://github.com/opentracing/opentracing-java)和[C++](https://github.com/opentracing/opentracing-cpp)。PHP和Ruby正在开发中。

## Data Conventions 数据约定
OpenTracing APIs的设计和规范，对追踪软件开发和探针软件开发都有通用指导意义；**追踪系统的开发者不必严格遵守指南，但是强烈推荐大家这么做。**

### Spans
1. Span命名
	在上一章OpenTracing interface中提到的Span interface包含operation name、tags、logs和baggage。这些可以代表span中进行的工作类型，例如：RPC或者HTTP调用端点、执行SQL语句，一个进程、类库或者模块名称。使得这个span所附带的信息是很有价值的。
2. Span结构
	Span结构也是非常重要的，主要表示Span本身所带重要信息，以及Spans之间的关联关系。后文具体介绍

#### Span Tag用例
Span Tag所携带的信息，特定存在的常量Key，有Errors、HTTP、Sample等等。不同的底层会对这些信息做一些其他处理。例如：获取、删除或者清空Span Tag下的所有信息，这个使用时可能是用得到的，但是需要注意的是，这些额外的操作可能会对业务系统本身造成不良影响，所以请仔细斟酌这些额外的实现。

#### Errors
表示一个span实例的错误状态，通过一个tag来标注。如果设置了一个span tag特定常量errors的值为true，这我们可以通过dashboard ui的tag查询功能，查询所有trace error列表，并通过tag的log日志，查看具体的web request错误信息，快速定位。并可以对一段时间的所有traces error列表统计，并报表输出。这个是非常重要的

#### Component Identification 组件定义
我们十分推荐库或者模块为监控程序提供组件的定义，最终用户可能会用拥有一个由框架和第三方混合提供的监控。例如：

1. span tag的key：`component`，value：需要被监控的类库、模块或者包的基本名称；例如：`httplib`、`JDBC`、`mongoose`等；
2. span tag的key：`span.kind`。value：`client | server`。指定这个span代表一个客户端或者服务端。


如果引入的package都自带分布式跟踪系统监控标签，则监控粒度就可以很容易的植入进来。

#### HTTP Server Tags
因为是http请求服务入口，所以这些都是在框架层做的事情。例如：

1. span tag的key：`http.url`；value：url地址；如：`http://www.google.com.hk`
2. span tag的key: `http.method`; value: `get|post|head`等
3. span tag的key：`http.status_code`; value: `200; 404; 503`等


#### Peer Tags
这个用于rpc服务使用，描述远程请求过程中，请求调用的方向。(客户端记录下行访问，服务端记录上行访问)。
1. `peer.hostname`目标主机名，类型：string
2. `peer.ipv4`目标IPv4地址，类型：string
3. `peer.ipv6`目标IPv6地址，类型：string
4. `peer.port`目标端口，类型：int
5. `peer.service`目标服务名称，string

peer翻译：对等，比如：client设置tag的peer{hostname, ipv4, service , port ...}，表示server的具体信息；server设置tag的peer{hostname...}, 表示client的具体信息。

#### Sampling采样
OpenTracing API不强调采样的概念。有些情况下，当业务量非常大时，如果业务使用了分布式追踪系统功能，则可能会对业务系统的性能造成很大影响。如果追踪系统产品本身实现了采样的特性，可以支持多规则采样，

1. 流量规则；
2. web request数量；
3. 预期的指定trace(特定的订单ID、支付ID、用户ID，预先植入，追踪特定请求trace)；

对于以上三种都是非常有价值的采样规则；span tag中有个采样key：`sampling.priority`, value值为整数类型；

1. value>0；追踪系统尽可能保留这条trace;
2. value=0；追踪系统不保存这条调用链;

如果此tag没有提供，追踪系统使用自己的默认采样规则；

### Logs
Logs日志是轻量级trace日志，与分布式日志系统无关；它是记录span事件日志，例如：当span执行单元发生错误时，错误日志存储在span logs中event：`error`, message： 具体的错误信息；

### Inject和Extract
在[《OpenTracing——相关概念术语》](https://gocn.vip/article/856)一文中提到，spans之间的trace信息携带，如果是是跨进程，需要把数据通过`Baggage`SpanContext携带，主要用于:

1. rpc服务;
2. 发布-订阅机制;
3. 通用消息队列;
4. HTTP请求调用;
5. UDP传输和其他传输方式；


把携带信息通过OpenTracing APIs中的Inject和Extract方法，在跨进程追踪时进行写入和读取。

`Inject`和`Extract`方式实现的设计，必须遵循以下要求：

1. 必须不需要使用OpenTracing使用中的特定代码；
2. 必须不需要针对每一种已知的跨进程通讯机制都处理；
3. 这套传播机制是最利于扩展的；

#### 基本方法：Inject、Extract和Carriers
追踪过程中的SpanContext可以被Inject方法注入到Carriers中，这个Carriers可以是一个`接口`或者一个`数据载体`。 并在跨进程间传输，OpenTracing标准包含两种必须的Carriers格式，自定义的Carriers也是可以的。同时在这个Span的ChildOf中通过Extract方法抽取Carriers信息，得到一个SpanContext。这个SpanContext表示被Inject到Carriers中的信息。

Inject伪代码:

```shell
span_context = ...

carrier = {} // 初始化数据载体
// 把span_context存放到HTTP_HEADERS格式的数据载体中
tracer.inject(span_context, opentracing.Format.HTTP_HEADERS, carrier)

// 并把carrier载体数据写入到跨进程调用client端的网络传输中。
for key, value in carrier:
	outbount_request.header[key] = value
```

Extrace伪代码：

```shell
inbound_request = ...


// 把跨进程服务端接收到的传输数据，按照指定的格式和字段，抽取到span_context中
carrrier = inbound_request.headers
span_context = tracer.extract(opentracing.Format.HTTP_HEADERS, carrier)

// 并把tracer串联起来，创建一个新的span
span = tracer.start_span("...", child_of=span_context)
```

Carrier格式：

所有的Carriers都有自己的格式。一般格式都以常量表达，例如：Binary, TextMap, HTTPHeaders常量或者字符串指定；另一些，则通过Carriers的静态类型指定。

自定义的Carriers格式：

分布式跟踪系统底层的实现可以采用自定义的Carriers格式，进行Inject和Extract操作。例如：
**ArrrPC private RPC SubSystem**，我们希望增加OpenTracing的数据在RPC请求过程中传输。伪代码：

```shell
span_context = ...
outbound_request = ...

try:
	// 尝试使用自定义的carrier格式，把span_context写入到数据载体中
	// 如果失败，直接使用标准的HTTP_HEADERS格式封装
	tracer.inject(span_context, arrrpc.ARRRPC_OT_CARRIER, carrier)
	
except opentracing.UnsupportedFormatException:

	carrier = {}
	tracer.inject(span_context, opentracing.Format.HTTP_HEADERS, carrier)

for key, value in carrier:
	outbound_request.header[key] = escape(value)
```


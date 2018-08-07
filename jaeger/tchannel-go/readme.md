# TChannel

TChannel是一个支持多路复用和帧协议的RPC框架。tchannel-go是这个协议的Go版本实现，包括Hyperbahn的客户端库。

如果你想要首先编写一个小型Thrift和TChannel服务，请查看[本指南](https://github.com/uber/tchannel-go/blob/dev/guide/Thrift_Hyperbahn.md)。如果你不喜欢这个指南，可以帮忙做一些[贡献](https://github.com/uber/tchannel-go/blob/dev/CONTRIBUTING.md).

## 总览

TChannel是一个网络协议，它支持：

- 一个request/response模型；
- 在同一个TCP socket上复用多个请求；
- 无序响应；
- 流请求和流响应；
- Checksummed帧；
- 任意负载的传输；
- 多语言的实现简单；
- 类似redis的高性能；

这个协议为IPC，意图运行在数据中心网络上。

## 协议

TChannel帧有个固定长度的头部和3个可变长度字段。底层协议没有给这些字段赋予含义，但是client/server实现使用第一个字段去表示RPC模型中唯一endpoint或者函数名称。接下来的两个字段可用于任意数据。对于这三个字段的使用，有些建议如下：

- URI path + HTTP method and headers as JSON + body
- Function name + headers + thrift/protobuf

上面两条建议，都是针对arg3个参数赋予了含义，

对于第一条建议：

1. arg1 为URI路径；
2. arg2 为HTTP的请求method和headers；
3. arg3 为HTTP的body数据

对于第二条建议：

1. arg1 为方法名；
2. arg2 为headers
3. arg3 为thrift/protobuf的序列化数据

注意，TChannel编码只支持UTF-8。如果你想要使用JSON，你需要在TChannel之外进行字符串化和解析。

这个设计支持高效路由和路由转发：routers需要解析第一个或者第二个字段，但是没有解析也能够转发第三个字段；

在这个系统中没有客户端和服务端之前的概念。每个TChannel实例能够发起和接收请求，只要求一个可以监听的唯一端口。这个要求可能未来会发生变化。

详见[protocol specification](http://tchannel.readthedocs.org/en/latest/protocol/)。

## 例子：

- [ping](https://github.com/uber/tchannel-go/blob/dev/examples/ping)。一个使用raw TChannel的ping/pong例子。
- [thrift](https://github.com/uber/tchannel-go/blob/dev/examples/thrift)。一个使用Thrift协议的client/server例子。
- [keyvalue](https://github.com/uber/tchannel-go/blob/dev/examples/keyvalue)。 具有单独server和client二进制的一个keyvalue Thrift服务

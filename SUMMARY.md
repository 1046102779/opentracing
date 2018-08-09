
# 分布式跟踪系统目录：

* [dapper](dapper/README.md)
    - [Dapper论文](dapper/dapper.md)
* [opentracing标准](opentracing标准/README.md) 
    - [概念和目标](opentracing标准/概念与目标.md)
    - [名词与术语](opentracing标准/名词与术语.md)
    - [APIs](opentracing标准/OpenTracing APIs.md)
    - [设计分布式跟踪系统的有效方法](opentracing标准/设计分布式追踪系统的有效方法.md)
* [trace产品对比](分布式跟踪系统——产品对比.md)
* [opentracing-go](opentracing-go/README.md)
    - [Tracer实例](opentracing-go/opentracer-go源码阅读一.md)
    - [上下文信息携带](opentracing-go/Propagation.md)
    - [日志](opentracing-go/Log.md)
* [basictracer-go](basictracer-go/README.md)
    - [Tracer实例](basictracer-go/tracer.md)
    - [Span对象](basictracer-go/span.md)
    - [事件与上下文信息携带](basictracer-go/event-propagation.md)
    - [span存储与pb协议](basictracer-go/spanrecorder.md)
    - [DEMO](basictracer-go/example.md)
* [appdash](appdash/README.md)
    - [Tracer与Span](appdash/tracer-and-span.md)
    - [annotations与事件](appdash/annotations-and-event.md)
    - [存储](appdash/store.md)
    - [近期存储与限制存储](appdash/recentstore-and-limitstore.md)
    - [反射](appdash/reflect.md)
    - [SpanRecorder与Collector](appdash/recorder-and-collector.md)
    - [OpenTracing部分支持](appdash/opentracing.md)
* [jaeger](jaeger/README.md)
    - [特性](jaeger/Features.md)
    - [jaeger的演变](jaeger/jaeger的技术演进之路.md)
    - [TChannel协议](jaeger/TChannel/README.md)
        * [协议标准](jaeger/TChannel/协议标准.md)
        * [Affinity](jaeger/TChannel/affinity.md)
        * [熔断器](jaeger/TChannel/熔断器.md)
        * [http与json](jaeger/TChannel/http-and-json.md)
        * [metrics](jaeger/TChannel/metrics.md)
        * [流控](jaeger/TChannel/流控.md)
        * [Thrift](jaeger/TChannel/thrift.md)
        * [keyvalue例子](jaeger/TChannel/keyvalue例子.md)
        * [tcp监听关闭处理方式](jaeger/TChannel/go1.5版本的一种tcp监听关闭处理方式.md)
    - [tchannel-go](jaeger/tchannel-go/README.md)
        * [channel](jaeger/tchannel-go/channel.md)
        * [PeerList](jaeger/tchannel-go/peer01.md)
        * [PeerList与Peer](jaeger/tchannel-go/peer02.md)
        * [RootPeerList](jaeger/tchannel-go/peer03.md)
        * [peerScore与idleSweep](jaeger/tchannel-go/peerScore-and-idleSweep.md)
        * [allchannel与subchannel](jaeger/tchannel-go/peerScore-and-idleSweep.md)
        * [connection](jaeger/tchannel-go/connection.md)
        * [inboundcall与outboundcall](jaeger/tchannel-go/OutboundCall-and-InboundCall.md)
        * [message exchange](jaeger/tchannel-go/message-exchange.md)
        * [arguments](jaeger/tchannel-go/arguments.md)
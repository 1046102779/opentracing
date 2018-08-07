在服务启动时，开启两个goroutine，一个为client，终端输入发出web请求；一个为http server，监听端口为8080。这里不用浏览器或者其他做client的原因是，需要client端创建trace，这样才能够在client/server跨进程形成调用链。 client直接回车，则结束进程。

在启动两个goroutine之前，初始化了global tracer, 并指定了Collector和Storage于一身的TrivialRecorder. 它的RecordSpan方法只是输出到终端的Span所有相关信息：操作名、创建时间、生命周期时间、日志列表长度、上下文携带信息和日志列表中的每行日志

其中：dapprish/random.go没有作用。

# Server

该服务启动了http的8080端口服务，接受所有请求，http服务的网络传输数据Carrier是采用的TextMapPropagation协议

它会在每个请求到来时，会根据Client的SpanContext信息创建一个名称为"ServerSpan"的Span。这个DEMO健壮性差，容易panic的原因在于：

`如果非下面的Client请求，而是其他web请求进来的，则不会带有Carrier的网络传输数据，则在TextMapPropagation的Extract解析时，报ErrSpanContextNotFound错误， 则在server中直接panic掉`

获取request body数据流后，通过Span.Logs记录请求数据，并通过span.Finish方法存储RawSpan数据={TraceID、SpanID、Sampled和Baggage}

# Client

client通过终端输入发送web http POST请求，它首先创建名称为"getInput"的Span， 并在span的logs中记录ctx、user text、返回response等内容；以及携带除了TraceID、SpanID、Sampled和Baggage{"User": 当前终端登录用户名}, 当span生命周期结束时，通过span.Finish方法存储RawSpan信息到内存中。


输入"hello,world"，执行结果如下所示：

```shell
Enter text (empty string to exit): hello,world
RecordSpan: serverSpan[2018-07-05 14:19:44.151121569 +0800 CST m=+7.236240362, 57.771µs us] --> 1 logs. context: {5671206282793949525 9024173529197870936 false map[user:chenchao]}; baggage: map[user:chenchao]
    log 0 @ 2018-07-05 14:19:44.151178152 +0800 CST m=+7.236296945: [request body:hello,world]
RecordSpan: getInput[2018-07-05 14:19:36.916512195 +0800 CST m=+0.001382988, 7.235279179s us] --> 3 logs. context: {5671206282793949525 978135273461507120 false map[User:chenchao]}; baggage: map[User:chenchao]
    log 0 @ 2018-07-05 14:19:36.916526835 +0800 CST m=+0.001397628: [ctx:context.Background.WithValue(opentracing.contextKey{}, &basictracer.spanImpl{tracer:(*basictracer.tracerImpl)(0xc4201340c0), event:(func(basictracer.SpanEvent))(nil), Mutex:sync.Mutex{state:1, sema:0x0}, raw:basictracer.RawSpan{Context:basictracer.SpanContext{TraceID:0x4eb42c111de59955, SpanID:0xd9308d94cf3e430, Sampled:false, Baggage:map[string]string{"User":"chenchao"}}, ParentSpanID:0x0, Operation:"getInput", Start:time.Time{wall:0xbec78bfe36a0ddc3, ext:1382988, loc:(*time.Location)(0x14960c0)}, Duration:7235279179, Tags:opentracing.Tags(nil), Logs:[]opentracing.LogRecord{opentracing.LogRecord{Timestamp:time.Time{wall:0xbec78bfe36a116f3, ext:1397628, loc:(*time.Location)(0x14960c0)}, Fields:[]log.Field{log.Field{key:"ctx", fieldType:10, numericVal:0, stringVal:"", interfaceVal:(*context.valueCtx)(0xc4200a6f00)}}}, opentracing.LogRecord{Timestamp:time.Time{wall:0xbec78c0008cf8373, ext:7232936124, loc:(*time.Location)(0x14960c0)}, Fields:[]log.Field{log.Field{key:"user text", fieldType:0, numericVal:0, stringVal:"hello,world", interfaceVal:interface {}(nil)}}}, opentracing.LogRecord{Timestamp:time.Time{wall:0xbec78c0009085b42, ext:7236661387, loc:(*time.Location)(0x14960c0)}, Fields:[]log.Field{log.Field{key:"response", fieldType:10, numericVal:0, stringVal:"", interfaceVal:(*http.Response)(0xc42018a090)}}}}}, numDroppedLogs:0})]
    log 1 @ 2018-07-05 14:19:44.147817331 +0800 CST m=+7.232936124: [user text:hello,world]
    log 2 @ 2018-07-05 14:19:44.151542594 +0800 CST m=+7.236661387: [response:&{200 OK 200 HTTP/1.1 1
1 map[Date:[Thu, 05 Jul 2018 06:19:44 GMT] Content-Length:[0]] {} 0 [] false false map[] 0xc42014c000
<nil>}]
```

# 总结

opentracing-go库是对OpenTracing标准的代码范式表达，而basictracer-go是对前者的最小子集扩展和实现，理论上这个basictracer-go已经可以进行业务开发和使用了，但是这个库还非常弱，主要表现在：collector和storage基于一身，且还是基于内存的，没有dashboard UI支持，无法用于生产。虽然无法用于生产，但是Span的实现，Carrier的支持已经做得很好了，同时这个库希望厂商采用OpenTracing标准实现时，可以把它作为中间层或者组件使用，因为它对厂商开放了一些接口自定义实现，例如：Carrier的AccessPropagation（IPC/RPC数据转换和存储），SpanRecord接口（RawSpan信息存储）。但是厂商也可以完全自己基于opentracing-go设计一套完整的分布式跟踪系统，不用再在opentracing-go和厂商中间加一层basictracer-go。

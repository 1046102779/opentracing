首先，我们先说一下，opentracing-language是支撑OpenTracing标准的最小子集，而basictracer-language是对这最小子集的基本实现；我们可以不使用basictracer，直接厂商作为最小子集的扩展就行；所以basictracer是其中一个扩展子集，方便大家做tracer设计参考；

# Tracer

basictracer的Tracer扩展了一些参数集options，支持以下操作：

1. ShouldSample函数：可以根据一定规则，对匹配到的tracer则发起跟踪；
2. TrimUnsampledSpans布尔值：当某个tracer被丢弃时，直接不会在这条链路上的各个span产生数据存储；
3. Recorder：当Span执行单元结束被Finish后，则形成特定span recorder并通过agent发送到collector存储；
4. NewSpanEventListener：trace事件监听，// ::TODO 后面更新
5. DropAllLogs布尔值： 如果NewSpanEventListener被设置了，则该布尔值变量无效；否则，如果为True，则所有的Span Logs都指向noop操作
6. MaxLogsPerSpan：因为span logs在OpenTracing标准中只是说减少Baggage信息存储，因为跨IPC/RPC网络延迟大，这里是指span本地logs存储尽量少，毕竟OpenTracing不是分布式日志管理系统。
7. EnableSpanPool布尔值：复用已经finish的span列表，减少内存使用；

默认的Options：

1. ShouldSample函数为mod 64，尽最大能力保存该条trace；
2. MaxLogsPerSpan值为100

创建Tracer实例的方法有两种：NewWithOptions和New。前者传入Options参数，最常用；后者使用默认的Options，传入Recorder来存储Tracer

## SpanContext Impl

basictracer中的SpanContext实现了opentracing-go中的SpanContext interface, 本地数据存储格式如下：

```shell
type SpanContext struct {
	TraceID uint64
	SpanID uint64
	Sampled bool
	Baggage map[string]string
}

func (c SpanContext) ForeachBaggageItem(handler func(k, v string) bool){
	for k, v := range c.Baggage{
		if !handler(k, v){
			break
		}
	}
}

func (c SpanContext) WithBaggageItem(key, val string) SpanContext{
	var newBaggage map[string]string
	if c.Baggage ==nil{
		newBaggage = map[string]string{
			key: val,
		}
	} else {
		newBaggage = make(map[string]string, len(c.Baggage)+1)
		for k, v := range c.Baggage{
			newBaggage[k] = v
		}
		newBaggage[key] = val
	}
	
	return SpanContext{c.TraceID, c.SpanID, c.Sampled, newBaggage}
}
```

上面的SpanContext实现了opentracing-go的SpanContext interface，并且提供了额外的操作和存储： 

1. TraceID、SpanID和Sampled，没有存储到Baggage的map中，而是单独拎出来，含义更加鲜明，易于理解；
2. 提供了SpanContext的WithBaggageItem方法，装载Baggage携带信息;

同时，这里的WithBaggageItem方法，我有**两个疑问**：

1. SpanContext的生命周期在于本地Span，在RPC/IPC时会通过TextMapCarriers、HTTPHeaderCarriers和BinaryCarriers进行携带，同一个Span执行单元不会并发，所以应该不需要新建一个newBaggage存储，写入然后再赋予给SpanContext中的Baggage；
2. 这个方法为何还要返回值SpanContext，如果不返回值，我觉得没什么问题；

## TracerImpl

tracerImpl实现了Tracer最基本的最小子集OpenTracing标准——opentracing-go；

```shell
type tracerImpl struct{
	// tracer的扩展参数集
	options Options
	// TextMap和HTTPHeader
	textPropagator *textMapPropagator
	// 比特流
	binaryPropagator *binaryPropagator
	// 如果厂商打算基于basictracer进行扩展，这里提供一个Baggage上下文信息传输的interface接口方法列表
	accessorPropagator *accessorPropagator
}
```

tracerImpl中的StartSpan, Inject和Extract方法，其中，后两个方法:

1. Inject方法根据RPC/IPC传输数据格式，来进行具体的Inject操作；如：format为TextMap或者HTTPHeader，则通过textMapPropagator进行SpanContext的数据转换；如果是binary，则通过binaryPropagator进行数据转换；如果是厂商自己实现，则通过accessPropagator委托实现；`notice: TextMap和HTTPHeader虽然存储数据格式不同，但是opentracing-go的TextMapReader和TextMapWriter已经屏蔽了数据格式存储方式`
2. Extract方法，同上，只是反方向，把RPC/IPC网络传输数据通过SpanContext数据存储格式进行数据转换；

对于StartSpan方法的代码片段，我有个疑问：

```shell
ReferencesLoop:
    for _, ref := range opts.References {
        switch ref.Type {
        case opentracing.ChildOfRef,
            opentracing.FollowsFromRef:

            refCtx := ref.ReferencedContext.(SpanContext)
            sp.raw.Context.TraceID = refCtx.TraceID
            sp.raw.Context.SpanID = randomID()
            sp.raw.Context.Sampled = refCtx.Sampled
            sp.raw.ParentSpanID = refCtx.SpanID

            if l := len(refCtx.Baggage); l > 0 {
                sp.raw.Context.Baggage = make(map[string]string, l)
                for k, v := range refCtx.Baggage {
                    sp.raw.Context.Baggage[k] = v
                }
            }
            break ReferencesLoop
        }
    }
```
以上我们可以看到，当前要创建的Span与其关联的References列表，只取出了首个Reference，且ParentSpanID的赋值，无论是ChildOf或者FollowsFrom都可以；我认为兄弟关系不可以成为ParentSpanID；

`备注：在SpanReferences列表中，只会存在[0,1]个ParentSpan和[0, N]个兄弟span`

在Options中有个属性变量：EnableSpanPool布尔值，如果为true，则使用sync.Pool临时对象池，当服务关闭后，则释放这些内存空间；它用来获取可用的Span，减少内存使用；

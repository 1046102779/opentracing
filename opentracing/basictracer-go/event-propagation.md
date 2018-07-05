# event

这个event是有关span操作时产生的事件，这些事件统称为：`SpanEvent`，相关事件数据存储格式如下：

```shell
// span创建时产生的事件
type EventCreate struct {OperationName string}

// 存储tag操作时的事件
type EventTag struct{
	Key string
	Value interface{}
}

// 存储baggage时的事件
type EventBaggage struct {
	Kye, Value string
}

// 存储日志时的事件
type EventLogFields opentracing.LogRecord

// span Finish时的事件
type EventFinish RawSpan

// 针对这些事件数据类型，有一些span操作事件的方法，如下所示：
func (s *spanImpl) onCreate(opName string) {
	if s.event !=nil {
		s.event(EventCreate{OperationName: opName})
	}
}

func (s *spanImpl) onTag(key string, value interface{}){
	if s.event !=nil {
		s.event(EventTag{Key: key, Value: value})
	}
}

func (s *spanImpl) onLogFields(lr opentracing.LogRecord){
	if s.event !=nil {
		s.event(EventLogFields(lr))
	}
}

func (s *spanImpl) onBaggage(key, value string) {
	if s.event !=nil {
		s.event(EventBaggage{Key: key, Value: value})
	}
}

func (s *spanImpl) onFinish(sp RawSpan) {
	if s.event!=nil {
		s.event(EventFinish(sp))
	}
}
// 以上这些都是针对span发生变化时的相关事件，这些可以记录，也可以不记录，取决于span.event是否被关注。
```

# propagation

propagation主要是IPC/RPC跨进程数据传输。它包括上下文信息携带的数据存储格式，以及网络传输数据与本地数据格式转换

我们在[《basictracer源码阅读——TracerImpl》](https://gocn.vip/article/868)的SpanImpl部分说过三种Baggage携带格式，包括TextMapPropagator、BinaryPropagator和AccessPropagator。

**这三种数据类型存在两种行为：Inject和Extract。用来进行SpanContext和Baggages之间的数据转换**

## textMapPropagator


```shell
// 父级：Client或者Producer
func (p *textMapPropagator) Inject(spanContext opentracing.SpanContext,
opaqueCarrier interface{}) error{
	sc, ok := spanContext.(SpanContext) // 获取span本身相关信息
	// 这个无需指明或者猜测是否为HTTPHeader或者TextMap，因为这是opentracing-go底层透明实现Set和ForeachKey两个方法
	carrier, ok := opaqueCarrier.(opentracing.TextMapWriter)
	
	// 把SpanContext需要上下文携带的信息写入到网络传输数据中
	carrier.Set(fieldNameTraceID, strconv.FormatUint(sc.TraceID, 16))
	carrier.Set(fieldNameSpanID, strconv.FormatUint(sc.SpanID, 16))
	carrier.Set(fieldNameSampled, strconv.FormatBool(sc.Sampled))
	
	// baggage信息, 在SpanImpl章节说过，为了凸显TraceID、SpanID和Sampled三个变量的价值，没有放入到Baggage中。
	for k, v := range sc.Baggage{
		carrier.Set(prefixBaggage+k, v)
	}
	return nil
}

// 子级：Server或者Consumer
func (p *textMapPropagator) Extract(opaqueCarrier interface{}) (opentracing.SpanContext, error) {
	// opaqueCarrier数据格式为HTTPHeader或者TextMap，这不用上层关心，因为这两者已经实现了ForeachKey方法
	carrier, ok := opaqueCarrier.(opentracing.TextMapReader)
	
	carrier.ForeachKey(func(k, v string) error{
		... // 根据Inject方法，解出对应的SpanContext相关信息
		// 包括TraceID、SpanID、Sampled和Baggage信息
	})
}
```

## binaryPropagator

它是网络传输比特流，通过io.Reader和io.Writer进行流读写操作

```shell
// 父级：Client或者Producer
func (p *binaryPropagator) Inject(spanContext opentracing.SpanContext,
opaqueCarrier interface{}) error{	
	// 获取span的SpanContext
	sc, ok := spanContext.(SpanContext)
	// 把SpanContext通过io.Writer, 采用protobuffer协议，
	// 把它转换为网络流中数据opaqueCarrier
	carrier, ok := opaqueCarrier.(io.Writer)
	
	// 然后把数据流通过pb协议和大端模式转化为网络流carrier
	state:=wire.TracerState{}
	state.TraceId = sc.TraceID
	state.SpanId = sc.SpanID
	state.Sampled = sc.Sampled
	state.BaggateItems = sc.Baggage // pb 支持map数据格式
	b, err :=proto.Marshal(&state)
	
	length:=uint32(len(b))
	binary.Write(carrier, binary.BigEndian, &length)
	carrier.Write(b)
	// 这里要注意的是，网络数据传输的格式是[length(b)+b]。读取的时候，前uint32八个字节为数据长度，偏移8字节后，计算uint32存储值，则是实际数据大小
	// 在linux内存管理基本上都是这种数据传输形式，例如： 存储struct{...}格式的数据流为：struct本身大小+struct中数据大小+struct数据体
}

// 子级：Server或者Consumer
func (p *binaryPropagator) Extract(opaqueCarrier interface{}) (opentracing.SpanContext, error) {
	// 获取binary网络流
	carrier, ok := opaqueCarrier.(io.Reader)
	binary.Read(carrier, binary.BigEndian, &length) //var length uint32
	buf := make([]byte, length) 
	// 以上我们可以看到，Inject注入到网络流的数据大小，Extract解析时也是一样的大小解析
	carrier.Read(buf)
	// buf是pb协议存储, 并通过pb协议解析
	ctx:=wire.TracerState{}
	proto.Unmarshal(buf, &ctx)
	return SpanContext{
		TraceID: ctx.TraceId,
		SpanID: ctx.SpanId,
		Sampled: ctx.Sampled,
		Baggage: ctx.BaggageItems,
	}
}
```

## accessPropagator

basictracer开放出了一个可供厂商自定义SpanContext与网络数据传输的存储协议。

```shell
type DelegatingCarrier interface{
	// 因为要遵循basicTracer的SpanContext，所以提供了SetState方法，读写traceID、SpanID和Sampled
	SetState(traceID, spanID uint64, sampled bool)
	State() (traceID, spanID uint64, sampled bool)
	// 再就是需要对SpanContext和Carrier进行数据转换
	// 这两个方法类似于TextMapReader和TextMapWriter接口
	SetBaggageItem(key, value string)
	GetBaggage(func(key, value string))
}

// 把SpanContext转换为网络可传输的Carrier，依赖于具体实现
func (p *accessorPropagator) Inject(spanContext opentracing.SpanContext,
carrier interface{}) error{
	// 转换SpanContext到Carrier中
	dc, ok := carrier.(DelegatingCarrier)
	sc, ok := spanContext.(SpanContext)
	dc.SetState...
	for k, v:= range sc.Baggage{
		dc.SetBaggageItem(k, v)
	}
}

// 把网络传输数据Carrier转换为SpanContext
func ( p *accessorPropagator) Extract(carrier interface{}) (opentracing.SpanContext, error) {
	// 网络数据校验是否为DeletingCarrier
	dc, ok := carrier.(DelegatingCarrier)
	// 获取Span相关信息
	traceID, spanID, sampled:= dc.State()
	sc := SpanContext{
		TraceID: traceID,
		SpanID: spanID,
		Sampled: sampled,
		Baggage: nil,
	}
	// 遍历获取baggage
	dc.GetBaggage(func(k, v string) {
		...
		sc.Baggage[k] = v
	})
	return sc, nil
}
```

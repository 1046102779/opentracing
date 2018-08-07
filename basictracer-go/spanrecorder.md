# SpanRecorder

SpanRecorder接口用于Span的存储，也就是担任Collector和Storage的职责。它向厂商开放了自定义实现自己的Collector和Storage服务或者组件；存储数据={TraceID, SpanID, Sampled, Baggage}；

```shell
type SpanRecorder interface {
	RecordSpan(span RawSpan)
}
```

basictracer-go内部已经实现了一个基于内存的SpanRecorder。SpanRecorder具体实现赋值是在初始化globaltracer时指定的，也即tracer.options.SpanRecorder变量值；

基于内存的SpanRecorder实现：

```shell
type InMemorySpanRecorder struct {
	sync.RWMutex
	spans []RawSpan
}

func NewInMemoryRecorder() *InMemorySpanRecorder {
	return new(InMemorySpanRecorder)
}

// 对RawSpan进行收集和存储工作
func (r *InMemorySpanRecorder) RecordSpan(span RawSpan) {
	r.Lock()
	defer r.Unlock()
	r.spans = append(r.spans, span)
}

// 获取[]RawSpan的快照，因为瞬时干净列表, 只是如果要并发获取[]RawSpan以及厂商实现不限制内存的话，则内存猛增。
func (r *InMemorySpanRecorder) GetSpans() []RawSpan {
	r.RLock()
	defer r.RUnlock()
	spans := make([]RawSpan, len(r.spans))
	copy(spans, r.spans)
	return spans
}

// 存在GetSpans和GetSampledSpans两种方法，主要是因为：
// Tracer和Span都存在Sampled，之间存在一定的关系
// 1. 当Tracer设置这个tracer需要尽量保存下来时，Tracer.Sampled=true，不用理会Span.Sampled值
// 2. 当Tracer.Sampled=false时，则可以根据span执行单元情况，选择是否要进行Span保留；
func (r *InMemorySpanRecorder) GetSampledSpans() []RawSpan {
	... // 所以这个是用来保留采样的Span, 与Tracer无关
}

// 清空[]RawSpan内存存储
func (r *InMemorySpanRecorder) Reset(){
	r.Lock()
	defer r.Unlock()
	r.spans = nil
}
```

## util

主要用来生成随机数， 赋值给TraceID和SpanID值;

## wire 

wire为binaryPropagator网络传输的协议数据流，该协议为google的protobuffer，在grpc中广泛使用。

```shell
// bp协议数据格式
message TracerState {
	fixed64 trace_id = 1;
	fixed64 span_id = 2;
	bool sampled =3;
	map<string, string> baggage_items = 4;
}
```

该协议的数据格式在[《basictracer-go源码阅读——event&propagation》](https://gocn.vip/article/875)的binaryPropagator中通过Inject和Extract方法说明过。

同时，TracerState也实现了DelegatingCarrier接口，这个在accessPropagator中给厂商开放了自定义实现Carrier的数据存储和转换；这里不再赘述了。

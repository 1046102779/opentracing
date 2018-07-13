说过几次Appdash的出生早于OpenTracing标准，为了支持OpenTracing标准，Appdash做了一些扩展，但是我觉得它并不遵循OpenTracing标准, 添加了一个库和简单的补充。用户既可以使用老版本的实现，也可以使用OpenTracing标准扩展的实现。

在Appdash源码有个opentracing目录，这个是扩展实现。

# OpenTracing

Appdash基于部分OpenTracing标准的扩展实现，它是采用了basictracer-go库进行兼容实现的。这样的改动代价是最小的。


## Tracer

因为Appdash扩展的OpenTracing标准实现，是直接在basictracer-go实现的基础上扩展的，所以相关的扩展都是针对basictracer-go的相关结构扩展

```shell
// 校验是否实现了interface
var _ opentracing.Tracer = NewTracer(nil)

// 在basictracer-go库中的Tracer参数，是采用的Options传递，所以Appdash的扩展也继承了这种方式便于赋值, Appdash中的Options相关参数在basictracer-go中都能找到对应的 tracer 参数。
type Options struct {
    ShouldSample func(traceID uint64) bool
    
    Verbose bool
    
    Logger *log.Logger
}

// 默认的Tracer采样是全量的, 每条tracer都尽量被存储。
func DefaultOptions() Options {
	return Options {
		ShouldSample: func(_ uint64) bool { return true },
		Logger: newLogger(),
	}
}

func NewTracer(c appdash.Collector) opentracing.Tracer {
	return NewTracerWithOptions(c, DefaultOptions())
}

func NewTracerWithOptions(c appdash.Collector, options Options) opentracing.Tracer {
	opts := basictracer.DefaultOptions()
	opts.ShouldSample = options.ShouldSample
	opts.Recorder = NewRecorder(c, options)
	return basictracer.NewWithOptions(opts)
}
```

说到这里，大家要想到一个问题，因为basictracer-go库是完全遵循OpenTracing标准的，但是Appdash的Span设计是没有遵循标准的。所以在RPC/IPC网络传输和本地RawSpan进行数据转换时，以及网络数据解析时，都应该会遇到解析和数据转换问题，所以Tracer的Inject和Extract需要重新实现，而不能使用basictracer-go库。但是这里并没有看到相关实现，所以这个问题后面再看。::TODO


## Recorder

appdash扩展的opentracing中，Recorder实现了basictracer-go库的Recorder interface

basictracer-go Recorder:

```shell
type SpanRecorder interface {
	RecordSpan(span RawSpan)
}

// basictracer-go库已经给出了一个实现InMemorySpanRecorder
```

这里Appdash也给出了一个实现Recorder

```shell
type Recorder struct {
	collector appdash.Collector
	logOnce sync.Once
	varbose bool
	Log *log.Logger
}

// 新建Recorder
func NewRecorder(collector appdash.Collector, opts Options) *Recorder {
	...
	return &Recorder{
		collector: collector,
		verbose: opts.Verbose,
		Log: opts.Logger,
	}
}

// 实现basictracer-go的Recorder interface
func (r *Recorder) RecorderSpan(sp basictracer.RawSpan) {
	... // 校验是否需要采样
	//	baisctracer-go中的RawSpan数据转换为Appdash中的Span数据
	spanID := appdash.SpanID {
		Span: appdash.ID(uint64(sp.Context.SpanID)),
		Trace: appdash.ID(uint64(sp.Context.TraceID)),
		Parent: appdash.ID(uint64(sp.ParentSpanID)),
	}
	
	// 存储span name的event
	r.collectEvent(spanID, appdash.Span(sp.Operation))
	
	// 存储span logs的event， 首先进行basictracer-go库的logs序列化, 然后打上log event
	for _, log := range sp.Logs {
		logs, _ := materializeWithJSON(log.Fields)
		// collectEvent主要工作：进行event的序列化为Annotations, 然后存储
		r.collectEvent(spanID, appdash.LogWithTimestamp(string(logs), log.Timestamp))
	}
	
	//  tags属于非event事件， 存储Tags
	for key, value := range sp.Tags {
		val := []byte(fmt.Sprintf("%+v", value))
		r.collectAnnotation(spanID, appdash.Annotation{Key: key, Value: val})
	}
	
	// Carrier携带信息存储
	for key, val := range sp.Context.Baggage {
		r.collectAnnotation(spanID, appdash.Annotation{Key: key, Value: []byte(val)})
	}
	
	// 存储span的timespan event
	approxEndTime := sp.Start.Add(sp.Duration)
	r.collectEvent(spanID, appdash.Timespan{S: sp.Start, E: approxEndTime})
}
```

通过RecorderSpan方法可以知道，执行basictracer.RawSpan的span信息存储，需要大量的TCP网络传输，且每个事件都需要，非常耗时，且对于collector服务的性能也会有较大的影响。所以最好的方式是使用Appdash中的**ChunkedCollector**，进行agent本地存储一定的span信息后，在一次通过tcp发送到Collector服务，进行存储。

## 总结

针对上面提到的问题，Appdash中扩展支持OpenTracing的实现，与自身Span信息结构有很大的不同，无法对SpanID和Annotations进行Inject和Extract，以及本地与Baggage数据转换等，所以如果采用OpenTracing标准，同时Collector还是基于原来的SpanID和Annotations存储，则需要进行Recorder层面的basictracer-go RawSpan信息与appdash Span信息转换，这个正是由SpanRecorder方法实现的。

我们由此可以知道，Appdash对OpenTracing标准的支持并不高，而且支持力度很弱，官方并没有对Appdash做了大改动，只是做了一个基于basictracer-go扩展的一个小功能，没有做很深入的工作。

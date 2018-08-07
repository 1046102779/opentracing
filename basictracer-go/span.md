# Span

span interface扩展了opentracing-go最小子集，新增了两个方法

```shell
type Span interface{
	opentracing.Span
	
	// 获取operation name
	Operation() string
	
	// 创建span的时间
	Start() time.Time
}
```

## Span Impl

spanImpl实现了Span interface， span的创建在[《basictracer源码阅读——TracerImpl》](https://gocn.vip/article/868)最后一说明，通过sync.Pool临时对象池创建小对象； 

大家如果想对sync.Pool有更深入的理解，可以看看[《真有趣达达写的slab》](https://github.com/funny/slab), 非常有意思，特别是无锁的小对象池。

```shell
type spanImpl struct {
	tracer *tracerImpl
	event func(SpanEvent)
	// span是否存在并发，取决于执行单元的粗细粒度，如果跨goroutine使用，则会存在并发
	sync.Mutex
	// 记录了OpenTracing标准中Span最小子集的所有信息
	raw RawSpan
	// 这个span因为在tracerImpl中设置了MaxLogsPerSpan,存储被丢弃的日志记录数
	numDroppedLogs int
}

type RawSpan struct {
	// Baggage信息，并且还携带了TraceID、SpanID和Sampled
	Context SpanContext
	
	// 父SpanID，如果当前Span为tracer的头结点，则ParentSpanID=0
	ParentSpanID uint64
	
	// Span的Operation name
	Operation string
	
	// 创建Span的时间
	Start time.Time
	
	// Span执行单元的时间长度
	Duration time.Duration
	
	// 本身Tags信息
	Tags opentracing.Tags
	
	// 日志事件记录列表
	Logs []opentracing.LogRecord
}

// 设置span的操作名称
func (s *spanImpl) SetOperationName(operationName string) opentracing.Span {
	... // 注意：并发上锁
}

// 当span不需要被记录时，判定是否要对span的tags进行本地存储
func (s *spanImpl) trim() bool{
	return !s.row.Context.Sampled && s.tracer.options.TrimUnsampledSpans
}

// 为span设置tags
// 注意：当为span存储sampling.priority时，这个值是存储在spanImpl的属性rawspan中的sampled值，所以它在basictracer实现中并没有写入tags；
// 另外对于basictracer的实现，值得说的一点是：
// 1. 在tracerImpl中的Options中已经通过ShouldSample函数指定该tracer是否被跟踪；
// 2. 同时, 无论是否有tracer前置条件，span本身都可以选择是否记录并存储
func (s *spanImpl) SetTag(key string, value interface{}) opentracing.Span{
	defer s.onTag(key, value)
	... // 注意：并发上锁
}

// 通过key-value键值对列表转换为[]log.Field, 并存储到spanImpl Logs中
func (s *spanImpl) LogKV(keyValues ...interface{}) {
	...
	// 当发生错误时，记录错误日志的错误信息和错误位置
	s.LogFields(log.Error(err), log.String("function", "LogKV")
}

// Span当发生日志记录时，格式如下：
log:
time: 2006-01-02 15:04:05  log.Fields: 
[ 
	{key: function, value: LogKV}	
	{key: error, value: "non-even keyValues len: 3"}
]
time: 2006-01-02 15:04:08 log.Fields:
[
	{key: update, value: record not found}
]

// 当span.raw.Logs列表长度大于等于tracer.Options.MaxLogsPerSpan(非零)时，这需要丢弃日志
// 这里丢弃日志的做法第一次见到，感觉还不错:
// 它是把一个列表中上半部分的日志反复覆盖写，这样的话，有个问题日志出现开头部分连续和结尾部分连续，中间断层。你可以通过问题发生的地方和结尾地方开始追踪，并在分布式日志管理系统中，根据trace的相关开始和结束日志，定位时间区域，和问题开始和结束的位置和原因等, 中间断层的日志在具体的日志系统中查找定位。
func (s *spanImpl) appendLog(lr opentracing.LogRecord) {
	...
	
	numOld:=(maxLogs -1 ) /2
	numNew = maxLogs - numOld
	s.raw.Logs[numOld+(s.numDroppedLogs%numNew)]=lr
	s.numDroppedLogs++
}

// 该方法记录一个错误发生时，相关的调用栈具体日志错误位置和错误信息
func (s *spanImpl) LogFields(fields ...log.Field) {
	...
}

func (s *spanImpl) Finish() {
	s.FinishWithOptions(opentracing.FinishOptions{}
}

func (s *spanImpl)FinishWithOptions(opts opentracing.FinishOptions) {
	...
	// 这里涉及到一个列表旋转的算法, 本节下面有旋转介绍和理解
	rotateLogBuffer(s.raw.Logs[numOld:], s.numDroppedLogs%numNew)
	// 做完之后，还要把span释放到sync.Pool临时对象池中复用 spanPool.Put(s)
}
```

**乍一看，卧槽，这个旋转有毛线用，再仔细理解理解appendLog方法；会再来一句，这想法牛逼；**

```shell
为啥牛逼呢？

因为appendLog方法在span.RawSpan.Logs溢出后，以后都会产生日志丢弃行为，而且是从后半段丢弃和重写，这样后半段反复覆盖重写，注意一点，我们在LogRecord存储时，需要保证日志输出的时间有序性，但是后半段反复覆盖写，则会导致一种情况：

当(s.numDroppedLogs)/numNew>1时，则会出现后半段的前N个日志时间戳大于后面的日志时间戳，这个分界线的位置则是：s.numDroppedLogs%numNew，所以需要这个左边位置向左旋转N。

这样的话，后半段日志记录时间戳就是有序递增的，那么输出也就是有序的
```

**spanImpl中的LogEvent, LogEventWithPayload, Log需要废弃，因为标准中opentracing-go的LogData已废弃，只保留LogKVs和LogFields方法。**


对于数组/列表旋转算法，可以采用C++中的官方算法库[rotate left](http://www.cplusplus.com/reference/algorithm/rotate/)，[《STL 源码分析》](https://blog.csdn.net/ww32zz/article/details/48995423)有助于这个算法的理解

这个算法是采用前向移动, 且每次移动[first, pos]大小固定数据单元

```shell
一维旋转，向(左|右)旋转N位的理解：

把一维数组分为两部分，则它当前由三部分构成：左半部分和右半部分；

记住一点：左半部分是整体、右半部分也是整体。

例如：字符串abcdefg ，向(左|右)旋转3位：
1. 向左旋转3位：
	整体可以看成：A'C'。其中：A'表示abc, C'表示defg， 则左旋转结果为：C'A', 展开结果为：efgdabc
2. 向右旋转3位：
	整体可以看成：A'C'。其中：A'表示abcd，C'表示efg，则右旋转结果为：C'A', 展开结果为：efgabcd
```

# 参考资料

[basictracer-go](https://github.com/opentracing/basictracer-go)

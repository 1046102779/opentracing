Appdash是基于basictracer-go扩展实现的，在此基础上丰富了的Collector和Storage。以及增加了dashboard UI 一些简单的查询统计功能。

Appdash中的Trace和Span struct与basictracer-go中的traceImpl和spanImpl struct的关系，后面再看。 **::TODO**


# trace

basictracer-go的扩展实现中，虽然是形成了完整的调用链，但是如果想要通过dashboard去进行绘图这条调用链, 则只能通过SpanRecorder的[]RawSpan，去通过算法找到一条完整的链路，可能这个操作还非常耗时。

所以Appdash针对Dashboard的业务场景，定义了基于Trace的显式链路结构：

```shell
// Appdash的Trace结构设计：采用了广度遍历思想，层次树存储。
// ChildOf和FollowsFrom理解：Sub列表为FollowsFrom关系，Trace和Sub列表为ChildOf关系
type Trace struct {
	Span // 根部span
	Sub []*Trace // child node
}

type Span struct {
	ID SpanID // SpanID= {TraceID, SpanID, ParentID}
	// Annotations携带了Span的所有信息；
	// 包括Tags，Baggage，Logs，OperationName等；
	
	// 其中span.Log进行分类，Appdash自身已定义了：五种Log
	// SpanNameEvent, logEvent, msgEvent, timespanEvent和Timespan
	// 类型分别是：name, log, msg, timespan和TimeSpan
	// 其中后两者的不同后面再理解 ::TODO
	// 对于Event也可以进行自定义实现， 但是Annotations不是直接的分类，需要对Annotations与Event进行数据转换存储才行
	Annotations // []{key:values}
}

// 对于span.Logs的类型定义，在Appdash中可以扩展，只需要实现Event interface， 并通过RegisterEvent方法注册自定义span logs事件
// Schema方法的返回值，作为Log的key类型值
// 需要说明的一点是：内部的五种类型Log，具有一定的标识前缀：
// SchemaPrefix = "_schema:"
type Event interface {
	Schema() string
}

func (t *Trace) String() string {
	... // 序列化Trace数据
}

func (t *Trace) TreeString() string {
	... // 打印树数据，以树结构的形式输出。
}

// 从根节点开始遍历递归查找SpanID，返回Trace
// 可能大家有个疑问：为何返回Trace，而不是Span？
// 因为Trace类型是层次树结构，所以每个节点都可以看成树节点，等同看的话，这就是Trace。实际上是Span节点，但是我们通过任意的子节点，可以获取到TraceID等调用链的全局信息。
func (t *Trace) FindSpan(spanID ID) *Trace {
	... 
}

func (t *Trace) TimespanEvent() (TimespanEvent, error) {
	... // 它是在当前调用链的span节点上，找到span logs中五类的TimespanEvent事件并返回
	// 这个过程在下节会详细介绍
}
```

**在Appdash span logs的分类与Span的Annotations的数据转换与存储比较复杂，我会单独一节讲解span log内部五类，并对数据转换与存储进行详细介绍。**

# span

span作为调用链树中的子节点，它的存储结构如下（Span携带的annotations下节再详细介绍）：

```shell
type Span struct {
	// SpanID在序列化时String，变成了"TraceID/SpanID/ParentID"
	ID SpanID
	
	Annotations
}

// 对span进行序列化，信息包括：TraceID/SpanID/ParentID，以及span的所有携带信息：Tags、Baggages、Sampled、OperationName和Logs等
func (s *Span) String() string {
	... // 通过json序列化
}

// 返回Span的OperationName，它是通过Annotations的key为"Name"获取。
func (s *Span) Name() string {
	...
}

// SpanID管辖Span本身的信息，与basictracer-go的设计雷同。表现在：
//   basictracer-go中RawSpan包括了SpanContext，它并没有把SpanID，TraceID、Sampled信息放在Baggage中，而是单独拎出来。
// 这里的Appdash Span设计把TraceID、SpanID和ParentSpanID没有放在Annotations信息中获取，也是单独拎出来，更加清晰直观，这个信息是必须的，非携带信息，而是Span自身信息。
type SpanID struct {
	Trace ID
	Span ID
	Parent ID
}

func (id SpanID) string {
	... // 序列化SpanID为"TraceID/SpanID/ParentSpanID"
	// 后面提到的tracesByIDSpan排序，则可以做到全局Span排序，且这个排序结果是有层次的
	// 做到TraceID有序，SpanID有序，且都是连续的，例如：
	// TraceID_01/SpanID_01/0
	// TraceID_01/SpanID_02/SpanID_01
	// TraceID_01/SpanID_03/SpanID_02
	// TraceID_02/SpanID_01/0
	// TraceID_02/SpanID_02/SpanID_01
	// ...
	// 我们可以看到前面三个为一个整体Trace调用链，后面两个为一个整体调用链，且全局有序，trace中的Span有序，这个设计很棒！
}

// 这个猜测是用来序列化span信息的，包括携带信息Annotations
for (id SpanID) Format(s string, args ...interface{}) string{
	args = append([]interface{}{id.String()}, args...)
	return fmt.Sprintf(s, args...)
}

func (id SpanID) IsRoot() bool {
	return id.Parent == 0 // 校验该span是否为根节点
}

// 封装SpanID，通过protobuffer协议网络传输数据流
func (id SpanID) wire() *wire.CollectPacket_SpanID {
	return &wire.CollectPacket_SpanID {
		Trace: (*uint64)(&id.Trace),
		Span: (*uint64)(&id.Span),
		Parent: (*uint64)(&id.Parent),
	}
}

// 把protobuffer协议网络流转换成SpanID数据
func spanIDFromWire(w *wire.CollectPacket_SpanID) SpanID {
	return SpanID{
		Trace: ID(*w.Trace),
		Span: ID(*w.Span),
		Parent: ID(*w.Parent),
	}
}

// 生成根节点的SpanID信息
func NewRootSpanID() SpanID {
	return SpanID{
		Trace: generateID(),
		Span: generateID(),
	}
}

// 生成子节点的SpanID信息
func NewSpanID(parent SpanID) SpanID {
	return SpanID {
		Trace: parent.Trace,
		Span:  generateID(),
		Parent: parent.Span,
	}
}
```


```shell
// 如果这个树类型结构是这样的，大家怎么看呢？
// ::TODO 后面再理解
type Span struct {
	ParentID SpanID
	SpanID SpanID
	TraceID TraceID
	Annotations []{key-values}
	Sub []*Span
}
```

在trace代码中有个sort.Interface的接口实现类型, 内部是快排算法。

```shell
// 排序算法：只需要满足三点，即可实现
// 1. 比较；2. 交换；3. 参与排序的元素数量。
type Interface interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}

type tracesByIDSpan []*Trace
// 这个类型是对Span列表进行ID排序，从小到大排列。这个意义后面再看，因为Trace树层次本身就是按时间排序的，看看这个ID uint64类型值的生成算法. 
// ::TODO
```

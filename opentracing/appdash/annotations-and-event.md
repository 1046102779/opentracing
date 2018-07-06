# annotations

Annotations用于存储Span携带信息，除了Span自身部分信息={TraceID、SpanID和ParentID}，通过Span存储Span自身信息和Annotations携带信息。

**这里要说明的一点，我觉得Appdash并没有把Span自身信息和其他信息区分好，但是OpenTracing官方的basictracer-go库的spanImpl区分比较好；因为前者Span的OperationName应该是属于Span本身的，但是在Appdash中被存储到Annotation["Name"]中；**

```shell
type Annotations []Annotation

type Annotation struct {
	Key string
	Value []byte
}

// 用于校验Annotation的Key值是否在已注册事件中是关键信息
func (a Annotation) Important() bool {
	... // 遍历获取所有已注册事件列表，校验每个事件是否为重要事件，并对获取到的重要事件
	// 通过event.Important方法获取关键字列表keys
	// 遍历keys并与a.Key值比较，如果相同，返回true；否则返回false
}

func (as Annotations) String() string{
	... // 序列化Annotations的键值对列表
}

func (as Annotations) schemas() []string {
	... // Annotations的所有key，并返回
	// 注意一点：Appdash内部的五种Event数据key都是带有前缀"_schema:"。内部标识
	// 其他都是外部自定义注册事件值
}

func (as Annotations) get(key string) []byte {
	... // 在Annotations中找到键为key的value值，并返回
}

func (as Annotations) StringMap() map[string]string {
	... // 把Annotations列表转换成map结构数据后，返回
}
// 这里有个疑问，为何Annotations不直接采用map[string][]byte结构存储，而是使用slice列表结构存储？
// 答案能想到的就是，map的相同key是会做覆盖处理，而slice结构是做append追加处理。
// 但是前面所看到的通过key，获取value，并没有考虑重复，所以还是有疑问。

func (as Annotations) wire() (w []*wire.CollectPacket_Annotation) {
	... // 遍历Annotations列表，把携带信息转换为protobuffer协议传输的数据信息
	// 注意一点：这里是使用的Annotations的快照存储，防止数据动态修改。这里是否有必要使用快照，后面再看.
}

func annotationFromWire(as []*wire.CollectPacket_Annotations) Annotations {
	... // 从protobuffer协议数据流中把网络数据转换为Annotations结构数据存储。
}
```

# event

```shell
// Span携带信息数据Annotations中，有一些是事件信息。
// Appdash内部自身定义了五种Event
// Schema方法用于返回事件的Key值。这个Key值代表事件名称
type Event interface{
	Schema() string
}

// 在事件列表中，哪些事件是值得关注的事件或者metrics
// Important方法返回一个列表值，这个后面再看
type ImportantEvent interface {
	Important() []string
}

// 由于Event事件数据是存储在Span的Annotations数据中，而Annotations并没有显式指明哪些是Event列表数据，所以就存在一对解析方法。类似于encoding/decoding
type EventMarshaler interface {
	MarshalEvent() (Annotations, error)
}

type EventUnmarshaler interface{
	UnmarshalEvent(Annotations) (Event, error)
}

// 标准化使用Marshal方法，把Event数据转换成Annotations
// 如果传入的参数不支持MarshalEvent，则使用默认的方法去Marshal event数据
// 简单的Event可以不用MarshalEvent，直接使用默认的
// 复杂的Event数据可以自定义实现MarshalEvent和UnmarshalEvent方法序列化和反序列化数据为指定的数据类型Annotations
// Appdash内部的五类Event，类型简单，使用默认的序列化就行了
func MarshalEvent(e Event) (Annotations, error) {
	// 先尝试校验Event是否实现了MarshalEventer接口，如果是的话，直接序列化
	if v, ok := e.(EventMarshaler); ok {
		as, err := v.MarshalEvent()
		...
		as = append(as, Annotation{Key: SchemaPrefix + e.Schema()}
		return as, nil
	}
	
	// 否则，使用默认的序列化方法
	var as Annotations
	flattenValue("", reflect.ValueOf(e), func(k, v string) {
		as = append(as, Annotation{Key: k, Value: []byte(v)})
	})
	as = append(as, Annotation{Key: SchemaPrefix+e.Schema()})
	return as, nil
}
```

MarshalEvent方法对于Annotations的Event存储理解很重要；它是解答**为何Span的Annotations是Slice存储key-value，而不是使用Map[string][]byte结构存储？**的关键。

针对这个问题，我还提了一个issue，最后自己解答了, [issue:206](https://github.com/sourcegraph/appdash/issues/206) 。 主要原因是作者希望Annotations存储Event事件时，希望与该事件相关的信息存储在一起连续，这样取数据时，可以以`_schema:xxx`为边界，获取与该事件相关的所有信息。这也就是为何对于事件要加一个前缀的原因。类似于分隔符。如果事件内部的连续信息使用了`_schema:xxx`前缀，则会导致事件信息的分裂。

在MarshalEvent方法中，都是先进行事件其他信息的序列化，然后再在这个单元结尾处添加一个`_schema:xxx`, 表示这个事件类型名称和结束符，且这个事件的value是空值。

另外一点，默认序列化方法是使用的flattenValue，它会采用递归算法对传入的event值进行操作，把这个事件的所有属性和子属性全部添加到Annotations中。


把Annotations中的所关注的事件，反序列化给传入的event值。
```shell
func UnmarshalEvent(as Annotations, e Event) error {
	aSchemas := as.schemas()
	// 获取Annotations所有的事件key列表， 并通过遍历与传入的Event值比较，如果找不到，返回错误
	// 和MarshalEvent类似
	// 先尝试校验Event是否支持反序列化，如果不支持，则使用默认的反序列化方法
	unflattenValue("", reflect.ValueOf(&e), reflect.TypeOf(&e), mapToKVs(as.StringMap()))
	return nil
}
```

对于unflattenValue方法是耗时操作，因为它是全局Annotations检索。我觉得最理想的方法是继续对Annotations的存储进行规范化，例如：把Annotations存储数据进行分类，分为Event、Log等类型。

## 内部Event

Appdash内部已存在五种Event，分别是SpanNameEvent，logEvent，msgEvent，timespanEvent和Timespan。它们都是先Event接口：Schema方法

```shell
// _schema:name
type SpanNameEvent struct {Name string }

func (SpanNameEvent) Schema() string { return "name"}

// 由此可以看到Span的OperationName存储在Event中，且最终通过MarshalEvent方法序列化存储在Annotations中
func SpanName(name string) Event {
	return SpanNameEvent{Name: name}
}

// _schema:msg
type msgEvent struct { Msg string}

func (msgEvent) Schema() string { return "msg" }

func Msg(msg string) Event {
	return msgEvent{Msg: msg}
}

// _schema:timespan
type timespanEvent struct {
	S, E time.Time
} 
func (timespanEvent) Schema() string {
	return "timespan"
}

func (ev timespanEvent) Start() time.Time {
	return ev.S
}
func (ev timespanEvent) End() time.Time {
	return ev.E
}

// _schema:Timespan
type Timespan struct {
	S time.Time `trace: "Span.Start"`
	E time.Time `trace: "Span.End"`
}

func (Timespan) Schema() string { return "Timespan"}

func (s Timespan) Start() time.Time {return s.S}
func (s Timespan) End() time.Time {return s.E}

上面的timespanEvent与Timespan两种事件的不同，我目前还不理解，::TODO

// _schema:log
type logEvent struct {
	Msg string
	Time time.Time
}

func Log(msg string) Event {
	return logEvent{Msg: msg, Time: time.Now()}
}

func LogWithTimestamp(msg string, timestamp time.Time) Event {
	return logEvent{
		Msg: msg,
		Time: timestamp,
	}
}

func (logEvent) Schema() string {
	return "log"
}

func (e *logEvent) Timestamp() time.Time {
	return e.Time
}
```

本章小结：

Appdash是早于OpenTracing标准出现的，所以它并没有遵循OpenTracing标准。虽然该trace的log部分使用了basictracer-go，但是其实可以取代不要它，它只是一个扩展。见 [issue 207](https://github.com/sourcegraph/appdash/issues/207)。

对于Span的Annotations，存储数据包含了所有的Event列表，Event与Annotations的数据转换(序列化和反序列化)， 通过MarshalEvent和UnmarshalEvent方法实现。

文中也解释了Annotations为何不使用Map[string][]byte存储，而要用slice结构存储数据。

这部分Annotations与Event的序列化和反序列化涉及到大量的reflect，在后面会有相应的分析。


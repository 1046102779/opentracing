 解析完Collector后，本节讲解Appdash的后端存储，目前Appdash支持的后端存储全部都是基于内存的存储，也可以自定义对接后端存储，比如：mysql、redis等，需要做一些Collect方法数据存储的适配工作{SpanID, Annotations}

# Store

为了丰富Collect服务本地存储和网络存储，提供了可自定义实现的Store interface；同时也提供了一些基于内存的本地存储localmemory等；

```shell
// SpanID与Annotations的数据存储
type Store interface {
	Collector
	
	Trace(ID) (*Trace, error)
}

// 在做Dashboard时的查询条件
// 支持基于时间范围和TraceID列表的查询操作
type TracesOpts struct {
	Timespan Timespan
	
	TraceIds []ID
}

// 查询接口，通过TraceOpts实现Trace条件搜索, 返回符合条件的Trace列表。
type Queryer interface {
	Traces(opts TracesOpts) ([]*Trace, error)
}

// Trace聚合数据, 数据通过Aggregator interface获取
type AggregateResult struct {
	RootSpanName string
	
	Average, Min, Max, StdDev time.Duration
	
	Samples int64
	
	Slowest []ID
}

// 聚合操作, 它也是指定时间范围内进行Trace数据聚合操作，返回聚合数据列表
// 注意一点：这里的时间范围是指：过去一段时间范围
type Aggregator interface{
	// Aggregate(-72*time.Hour, 0)
	Aggregate(start, end time.Duration) ([]*AggregateResult, error)
}
```
## 持久化存储

持久化存储也就是把数据最终存储到磁盘上。PersistentStore interface是表示持久化存储，它在Store interface的基础上新增了ReadFrom和Write方法，用于持久化存储。

Appdash持久化存储的实现包括：MemoryStore

```shell
type PersistentStore interface{
	Write(io.Writer) error
	ReadFrom(io.Reader) (int64, error)
	Store
}
```

我们知道，无论是业务微服务的agent采集，或者Collector服务端，都是已Recorder单元进行数据处理，并没有形成完整的trace调用链。所以Storage首先就是要把离散的Span，聚合成一条条完整的trace列表。这样在Dashboard中显示时才能够形成调用链列表。那么在Store处理数据过程中，就会有插入分裂节点操作等

### MemoryStore

```shell
type MemoryStore struct {

	trace map[ID]*Trace // trace ID -> trace tree
	
	span map[ID]map[ID]*Trace // trace ID -> span ID -> sub trace tree
	
	sync.Mutex // 并发操作
}
```

这个MemoryStore比较有意思，Trace类型我们在[《Appdash源码阅读——Tracer&Span》](https://gocn.vip/article/879)文章中介绍过，它是一个Trace调用链完整的层次树。那么通过trace的map结构，我们可以通过TraceID获取整颗完整的trace树，这个map结构的trace存储的是一颗颗完整的trace树。

span的map结构存储的数据是通过TraceID可以获取到每个子节点的树，也就是把上一个map结构的trace展开存储，后者只能通过递归形式获取到某个span节点，但是这个map结构的span可以获取到全局到任一span节点，也就是说查询到的TraceID列表各个span的时间复杂度o(1)。 而前者是logN。

所以trace和span拥有的span节点是相同的，也就相当于2倍Trace存储。

上面这个结构设计，不太合理。

我们再看MemoryStore的创建、节点插入和读取等操作

```shell
// 创建一个MemoryStore实例
func NewMemoryStore() *MemoryStore {
	return &MemoryStore{
		trace: map[ID]*trace{},
		span: map[ID]map[ID]*trace{},
	}
}

// 这里提一个小技巧，如果我们想确认自定义的后端存储是否实现了Store、或者PersistentStore接口，可以在编译时确定，做法如下, 这种做法非常常见

var _ interaface {
	Store
	Queryer
} = (*MemoryStore)(nil)

// 存储Recorder
func (ms *MemoryStore) Collect(id SpanID, anns ...Annotation) error {
	ms.Lock()
	defer ms.Unlock()
	return ms.collectNoLock(id, anns...)
}

// recorder数据转换存储到MemoryStore中
func (ms *MemoryStore) collectNoLock(id SpanID, anns ...Annotation) error {
	// 增加span节点
	if _, present := ms.span[id.Trace]; !present {
		ms.span[id.Trace] = map[ID]*Trace{}
	}
	
	// insert || update span
	s, present := ms.span[id.Trace][id.Span]
	if !present {
		s = &Trace{
			Span: Span{
				ID: id,
				Annotations: anns,
			},
		}
		ms.span[id.Trace][id.Span] = s
	} else {
		s.Annotations = append(s.Annotations, anns...)
		return nil
	}
	
	// insert || update trace tree
	root, present := ms.trace[id.Trace]
	if !present {
		ms.trace[id.Trace] = s
		root = s
	}
	
	// 后面一段代码做了整个层次树的重建。
	// 做法：
	// 1. 如果当前trace的TraceID已存在，不管目前存储的trace tree根节点是否为真正的root节点； 分三种情况：
	//    (1). 新来的span是真正的根节点
	//    (2). 当前根节点的父亲是新来的span
	//    (3). 当前根节点的父亲不是新来的span
	// 前两者归为一类, 都把新来的span作为根节点处理
	// 对于(1), (2)表明已存在Trace，且新来的span是根节点的父亲，则做树的重建, 新的root为span
	if ... {
		// 改变根节点为新的span
		ms.trace[id.Trace] = root
		// 调整原来合适但不正确的depth=2的子节点，到新的root下。
		ms.reattachChildren(root, oldRoot)
		// 调整MemoryStore下的span
		ms.insert(root, oldRoot)
		
		
	}
	// 2. 当前trace不存在和(3)点归为一类：
	//    不做处理
	
	// 对于这个算法，后面有详细介绍，便于大家理解
	...
	
	// 存储span到MemoryStore的span合适位置上
	if !id.IsRoot() && s != root {
		ms.insert(root, s)
	}
	
	// 调整原来root下depth=2的子节点
	if s != root {
		ms.rettachChildren(s, root)
	}
	return
}

// 插入span到ms的span存储中
func (ms *MemoryStore) insert(root, t *Trace) {
	p, present := ms.span[t.ID.Trace][t.ID.Parent]
	if present {
		// 如果能够找到父亲，则直接添加在父亲的子节点列表中
		p.Sub = append(p.Sub, t)
	} else {
		// 如果不能找到父亲，则直接添加到root的子节点中，暂存。
		root.Sub = append(root.Sub, t)
	}
}

// 调整原来root的depth=2相关子节点
// 由于dst和src已在前面建立好了关系，所以这里只是对depth=2的子节点进行适当调整
// dst与src的链条不会变化
func (ms *MemoryStore) rettachChildren(dst, src *Trace) {
	var sub2 []*Trace
	
	for _, sub := range src.Sub {
		if sub.Span.ID.Parent == dst.Span.ID.Span {
			// 该节点和dst是父子关系，直接提上去
			dst.Sub = append(dst.Sub, sub)
		} else {
			sub2 = append(sub2, sub)
		}
	}
	// 因为src下面子节点已发生改变，所以直接赋予新的子节点列表即可
	src.Sub = sub2
}
```

针对collectNoLock方法，有个疑问。我们看到首先对span进行存储操作，当发现span节点已存在时，在追加Annotations后就直接返回了，但是trace的span子节点并没有追加Annotations。所以会导致trace和span的子节点数据不一致。

在collectNoLock后面代码对trace层次树进行重建的原因在于：

并不是创建第一个trace id时，它就是根节点，它也可能在网络中滞留了，导致span子节点先到，而root节点后到，如果不重建，则会导致调用链的乱序，或者无序。这颗树的重建依赖于各个span的parentspanid值

我们要理解层次树的重建，要搞清楚几点：

1. 重建后的树也不一定是有序的层次树；
2. 每次重建后，如果新来的span尚不能放在正确的位置，对于MemoryStore的trace存储，span就放在临时根节点的子节点上，距离为1.也就是root.Sub列表中。
3. 每次重建后，还不能放在正确位置的span，对于MemoryStore的span存储，span就存放在根节点的Sub列表中

**所以对于已存在的trace，如果新到来的span还不能存放在正确的位置，那就存放在depth=2的节点上。**

再加上一些对于MemoryStore的方法列表，包括查询、删除等操作

```shell
// 根据TraceID，获取MemoryStore中trace对应ID的调用链
func (ms *MemoryStore) Trace(id ID) (*Trace, error) {
	ms.Lock()
	defer ms.Unlock()
	return ms.traceNoLock(id)
}

func (ms *MemoryStore) traceNoLock(id ID) (*Trace, error) {
	t, present:= ms.trace[id]
	...
}

// MemoryStore在开头说到编译时也校验，它实现了Queryer接口
// 这里虽然实现了Traces查询功能，但是并没有使用输入参数。这个是全局输出所有的调用链
func (ms *MemoryStore) Traces(opts TraceOpts) ([]*Trace, error) {
	ms.Lock()
	defer ms.Unlock()
	
	for id := range ms.trace {
		t, err := ms.traceNoLock(id)
		ts = append(ts, t)
	}
	...
}

// 删除MemoryStore中的trace和span
func (ms *MemoryStore) Delete(traces ...ID) error {
	ms.Lock()
	defer ms.Unlock()
	return ms.deleteNoLock(traces...)
}

func (ms *MemoryStore) deleteNoLock(traces ...ID) error {
	for _, id := range traces {
		delete(ms.trace, id)
		delete(ms.span, id)
	}
	return nil
}

// 删除MemoryStore中的span，以及对应trace中的span
// 这里存在一个问题，为何有这种删除完整调用链的需求？只删除Annotations，我还可以理解。
// 如果删除Span，则会导致这个Span的所有子节点全部失联的情况。
func (ms *MemoryStore) deleteSubNoLock(s SpanID, annotationsOnly bool) bool {
	... 
}

// 因为MemoryStore实现了持久化存储PersistentStore, 所以存在io.Writer和io.Reader操作
// 使用gob进行序列化写入
func (ms *MemoryStore) Write(w io.Writer) error {
	... //并发
	
	data :=memoryStoreData{m.trace, m.span}
	return gob.NewEncoder(w).Encode(data)
}

// 从io流读取数据到内存MemoryStore中
func (ms *MemoryStore) ReadFrom(r io.Reader) (int64, error) {
	... // 并发
	
	var data memoryStoreData
	gob.NewDecoder(r).Decode(&data)
	ms.trace = data.Trace
	ms.span = data.Span
	return int64(len(ms.trace)), nil
}
```



从Appdash中，我们可以看到对于并发操作，都会存在两个方法一个是XXX(), 另一个则是XXXNoLock() ，如果存在并发操作，则使用前者，如果非并发，则使用后者。

另一个场景则是，对于mysql事务处理，在业务中也可以提供两类操作，一类是带事务，一类是不带事务操作。


## PersistentStore

持久化存储操作，前面提到如果要内存数据持久化到本地，则需要实现PersistetStore接口，内存本地化操作如下：

```shell
// 通过实现PersistentStore接口的Storage，通过对应的接口把io流定时写入到文件中。
// 这个是对全局调用链信息的持久化，定期全局覆盖备份, 在没有完成备份前，会写入到临时文件中
func PersistentEvery(s PersistentStore, interval time.Duration, file string) {
	for {
		time.Sleep(interval)
		
		f, err := ioutil.TempFile("", "appdash")
		
		s.Write(f)
		s.Close()
		os.Rename(f.Name(), file)
	}
}
```


## RecentStore

这两者有时间，我再介绍。

## LimitStore


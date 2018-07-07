# Recorder

在Appdash中，Recorder是可以与Collector交互，以及Span对外的操作入口或者操作单元。

```shell
type Recorder struct {
	SpanID
	annotations []Annotation
	
	finished bool // 表示该span是否已经结束了生命周期
	
	collector Collector // Recorder作为Collector的输入
	
	errors []error // 在span的生命周期内操作出现错误的error列表
	errorsMu sync.Mutex // 并发errors
}

// 当StartSpan后，再指定Collector，用于存储trace数据
// 这里有个疑问：为何Collector不是在各个服务启动时，初始化global tracer时并指定Collector服务，而是把Collector细化到Recorder中，这样如果服务在运行中，Collector不变，则直接提到Tracer上就可以了，如果经常变，那么Tracer调用链肯定是不会完整的，各个storage也不能正确的显示完整的调用链。后面再看 ::TODO
func NewRecorder(span SpanID, c Collector) *Recorder {
	...
	return &Recorder{
		SpanID: span,
		collector: c,
	}
}

// 根据当前Span，创建一个子Span。操作入口是Recorder级别
func (r *Recorder) Child() *Recorder {
	return NewRecorder(NewSpanID(r.SpanID), r.collector)
}

// Recorder操作级别，为Span添加OperationName，并通过Event与Annotations之间的序列化进行数据转换存储
func (r *Recorder) Name(name string) {
	r.Event(SpanNameEvent{Name: name})
}

// 同上，Event为msgEvent
func (r *Recorder) Msg(msg string) {
	r.Event(msgEvent{Msg: msg})
}

// 同上，Event为logEvent
func (r *Recorder) Log(msg string) {
	r.Event(Log(msg))
}

// 同上，只是增加带有时间戳
func (r *Recorder) LogWithTimestamp(msg string, timestamp time.Time) {
	r.Event(LogWithTimestamp(msg, timestamp))
}

// 把Event转换为Recorder中的Annotations
func (r *Recorder) Event(e Event) {
	ans, err := MarshalEvent(e)
	... 
	r.annotations = append(r.annotations, ans...)
}

// Recorder操作粒度，Span生命周期结束, 并把Recorder推送到Collector。
// 这里会有个疑问：如果每个Recorder在结束时立即推送，如果业务量爆发，则这个短连接的高并发推送，会对业务造成明显抖动。所以最好是长连接池
func (r *Recorder) Finish() {
	if r.finished {
		... // error
	}
	r.finished = true
	r.Annotation(r.annotations...)
}

func (r *Recorder) Annotation(as ...Annotation) {
	r.failsafeAnnotation(as...)
}

// 推送Span的所有信息到Collector中，包括自身信息和其他信息，这个量在网络传输中比较大，没有限制大小
func (r *Recorder) failsafeAnnotation(as ...Annotation) {
	return r.collector.Collect(r.SpanID, as...)
}

// 还有几个errors的并发处理，不展开了。
```

从上面可以了解到，Appdash中span的操作粒度为Recorder。通过Recorder来完成Span生命周期的所有操作。

# Collector

collector server限制了处理每个请求的数据大小`maxMessageSize=1M`，Appdash指定Collector server责任是处理Span Recorder event数据，但是业务微服务发过来的Recorder是包含了SpanID和Annotations所有数据，而event只是Annotations中的一部分数据，可能event占比很大。

```shell
// Collector可供用户自定义实现自己的Collector数据处理
type Collector interface {
	Collect(SpanID, ...Annotation) error
}

// Appdash的Collector和Storage设计思路为：
// 每种后端存储方式都会有对应的Collector数据处理方式。
// Collector是接收和清洗，Storage是存储。
// 下节我会详细介绍Appdash自带的多种内存存储方式
type Store interface {
	Collector
	Trace(ID) (*Trace, error)
}

// 本地存储，表示Collector服务和trace数据存储在同一个节点上
func NewLocalCollector(s Store) Collector {
	return s
}

// 这个表示Collector服务和Storage服务是分离的，当Collector服务完成数据接收、清洗后，再通过protobuffer协议传输到Storage服务进行数据存储。
func newCollectPacket(s SpanID, as Annotations) *wire.CollectPacket {
	return &wire.CollectPacket{
		Spanid: s.wire(),
		Annotation: as.wire(),
	}
}
```
Appdash的agent包含两类：一类是基于Recorder的块传输，一类是基于Recorder的单一传输。当业务对时延要求比较高时，同时能够忍受一定的时效性，则可以使用块传输ChunkedCollector；当业务对时延要求不太高，且希望时效性尽量好时，则使用RemoteCollector。


## Agent——ChunkedCollector
```shell
// ChunkedCollector是属于业务微服务的agent，由它传输到Collector server。
// Appdash提供了另一种比较好的agent模式， 场景在于如果微服务数量多，且业务量大，同时如果业务对时延性要求高，所以为了减少对业务造成的负载压力，我们可以先在业务微服务先暂存下来，在进行块传输。
// 这个类似于日志文件分割：希望日志文件能够按照时间和大小两个维度来分割。
// 1. 当日志文件流写入到一定时间时，直接Flush文件并关闭且重新打开新的文件描述符fd；
// 2. 如果没有到时间，日志文件流大小到达指定时，则也做关闭并打开新的fd；
type ChunkedCollector struct {
	Collector
	
	MinInterval time.Duration // 时间维度
	
	// 这个是表示如果写入超时，则直接丢弃剩下没有传输的数据
	FlushTimeout time.Duration 
	
	// 默认大小：32M
	// 这个变量代表的意义要仔细理解。
	// 当前ChunkedCollector数据内存块queueSizeBytes没有超过这个值时，如果下一个到来的Recorder累加，超过这个最大值，则直接抛弃接收到的Recorder，不会丢弃整个内存块。直到最小时间到来时Mininterval，写入后才能继续接收Recorder。这样的问题在于，如果MaxQueueSize设置不合理，同时Mininterval设置又很大的情况下，则会造成某一段时间Recorder捕获不到。
	// 简言之：溢出则直接丢弃新来的Recorder.
	MaxQueueSize uint64
	
	started, stopped bool
	stopChan chan struct{}
	
	queueSizeBytes uint64 // 当前队列大小
	pendingBySpanID map[SpanID]Annotations // 收集多个Span
	
	mu sync.Mutex // 并发操作
}

func NewChunkedCollector(c Collector) *ChunkedCollector {
	return &ChunkedCollector{
		Collector: c,
		MinInterval: 500*time.Millisecond, // 500ms写一次
		FlushTimeout: 2*time.Second, // 2秒没写入成功，直接丢弃
		MaxQueueSize: 32*1024*1024, // 32M
	}
}


// 这个Collector块大小处理写得不错，
// 存在一个问题，溢出时丢弃Span，可能使得某些trace调用链不完整。
func (cc *ChunkedCollector) Collect(span SpanID, anns ...Annotation) error {
	... // 并发操作，以及校验ChunkedCollector是否已停止处理
	if !cc.Started {
		cc.start() // 启动一个goroutine，用于定时出发pendingBySpanID的Flush Chunked操作。
	}
	
	// 计算ChunkedCollector的累积块大小
	// Recorder由两部分构成：SpanID和Annotations，其中SpanID={SpanID、TraceID和ParentID} 都是uint64 8个字节，所以SpanID共占用24个字节
	// 然后再累计Annotations字节数
	var collectionSize uint64 = 3*8
	for _, ann:= range anns {
		collectionSize += uint64(len(ann.Key))
		collectorSize += uint64(len(ann.Value))
	}
	cc.queueSizeBytes += collectionSize
	
	// 溢出则直接丢弃Recorder
	if cc.MaxQueueSize!=0 && cc.queueSizeBytes+collectionSize > cc.MaxQueueSize {
		return ErrQueueDropped
	}
	
	if p, present := cc.pendingBySpanID[span]; present {
		cc.pendingBySpanID[span] = append(p, anns...)
	} else {
		cc.pendingBySpanID[span] = anns
	}
	
	return nil
}

// 我觉得Appdash的业务微服务agent处理，会存在并发大内存拷贝现象，会很吃内存
func (cc *ChunkedCollector) Flush() error {
	start:=time.Now()
	cc.mu.Lock()
	... // 内存 拷贝内存块，释放锁，交给agent继续处理Recorder数据
	cc.mu.Unlock()
	
	// 省略了一些代码
	for spanID, p := range pendingBySpanID {
		cc.Collector.Collect(spanID, p...)
		
		if cc.FlushTimeout !=0 && time.Since(start) > cc.FlushTimeout {
			break
		}
	}
	...
	return nil
}

// 启动一个goroutine，用于agent传输内存块到Collector server
func (cc *ChunkedCollector) start() {
	cc.stopChan = make(chan struct{})
	cc.started = true
	go func() {
		for {
			t:= time.After(cc.MinInterval)
			select {
				case <-t:
					cc.Flush
				case <-cc.stopChan:
					return // stop
			}
		}
	}()
}

func (cc *ChunkedCollector) Stop() {
	...
	close(cc.stopChan)
	cc.stopped = true
}
```

这个Flush方法负责处理传输内存块到Collector server中， 我们看到代码中有个变量值cc. FlushTimeout， 当agent传输块内存到Collector server中超时时，则会直接丢弃没有传完的数据。这样做的目的是，防止agent卡死，因为agent的接收、处理和传输Span数据，是存在并发操作，且共用cc.mu.Lock锁，如果传输一直卡着，则agent接收Recorder也一直卡着。

## Agent——Remote Collector

```shell
// RemoteCollector两种tcp client连接，带证书和不带证书
func NewRemoteCollector(addr string) *RemoteCollector {
	reutrn &RemoteCollector{
		addr: addr,
		dial: func(net.Conn, error) {
			return net.Dial("tcp", addr)
		},
	}
}

func NewTLSRemoteCollector(addr string, tlsConfig *tls.Config) *RemoteCollector {
	return &RemoteCollector{
		addr: addr,
		dial: func() (net.Conn, error){
			return tls.Dial("tcp", addr, tlsConfig)
		},
	}
}

type RemoteCollector struct {
	addr string
	
	dial func() (net.Conn, error) 
	
	mu sync.Mutex
	pconn pio.WriteCloser // client connection
	
	... 
}

// 发送Span数据到Collector server， 传输数据序列化协议protobuffer
// 并通过rc.collect方法进行pconn.WriteMsg方法进行写操作
func (rc *RemoteCollector) Collect(span SpanID, anns ...Annotation) error {
	return rc.collectAndRetry(newCollectPacket(span, anns))
}

// pb协议，创建Collector client连接
func (rc *RemoteCollector) connect() error {
	c, err := rc.dial()
	
	rc.pconn = pio.NewDelimitedWriter(c)
}
```

## Collector Server

Appdash把Collector Client和Server都放在了collector.go文件中，感觉还是有些杂，如果能写为collector_client.go、collector_server.go和collector.go，就非常好了

Collector Server主要就是启动一个collector服务，监听并收集来自各个业务微服务中的agent发送过来的Recorder数据。

```shell
// 创建一个Collector Server。该方法的第二个参数Collector，是带有指定Remote/Local Storage的client。这个Collector实现了Store interface
// 第一个参数一般都是tcp长连接, 因为SpanID数量太多，如果HTTP请求，则三次握手消耗太大，无法接受
func NewServer(l net.Listener, c Collector) *CollectorServer {
	cs :=&CollectorServer{c: c, l: l}
	return cs
}

type CollectorServer struct {
	c Collector
	l net.Listener
}

// 启动Collector服务，并监听获取和处理来自agent的请求
func (cs *CollectorServer) Start() {
	for {
		conn,err := cs.l.Accept()
		
		go cs.handleConn(conn)
	}
}


// Collector服务处理方式和http一样，都是来一个请求，创建一个goroutine。
// 同时利用pb协议，读取io流，每次读取一个Recorder大小，并把Recorder发送给Store interface处理。
func (cs *CollectorServer) handleConn(conn net.Conn) (err error) {
	defer conn.Close()
	
	rdr := pio.NewDelimitedReader(conn, maxMessageSize)
	defer rdr.Close()
	for {
		p := &wire.CollectPacket{}
		rdr.ReadMsg(p)
		
		spanID := spanIDFromWire(p.Spanid)
		
		cs.c.Collect(spanID, annotationsFromWire(p.Annotation)...)
	}
}
```

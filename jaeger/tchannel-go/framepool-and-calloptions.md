# frame pool interface

frame pool提供了创建Frame的临时对象池，包括Frame的分配和回收，节约内存。

```shell
// frame pool的分配和回收interface
type FramePool interface {
	Get() *Frame
	
	Release(f *Frame)
}
```

tchannel提供了三种对frame pool interface的实现, 分别是disable、sync和channel，默认frame pool实例为sync.Pool

## disabled frame pool

disabled含义为关闭，没有使用临时对象池

```shell
var DisabledFramePool = disabledFramePool{}

type disabledFramePool struct{}

func (p disabledFramePool) Get() *Frame {
	return NewFrame(MaxFramePayloadSize)
}

func (p disabledFramePool) Release(frame *Frame) {}
```

## sync frame pool

协议帧实例默认使用sync.Pool临时对象池

```shell
var DefaultFramePool = NewSyncFramePool()

type syncFramePool struct {
	pool *sync.Pool
}

// 通过sync.Pool对象池创建Frame实例
func NewSyncFramePool() FramePool {
	return &syncFramePool {
		pool: &sync.Pool{
			New: func() interface{} {
				return NewFrame(MaxFramePayloadSize)
			},
		},
	}
}

// 从sync.Pool临时对象池获取一个frame
func (p *syncFramePool) Get() *Frame{
	return p.pool.Get().(*Frame)
}

// 把frame释放到sync.Pool临时对象池中
func (p *syncFramePool) Release(f *Frame) {
	return p.pool.Put(frame)
}
```

## channel frame pool

使用channel，在队列上存储指定数量的Frame

```shell
type channelFramePool chan *Frame

// 新建channel队列数量为capacity数量的frame
func NewChannelFramePool(capacity int) FramePool {
	return channelFramePool(make(chan *Frame, capacity))
}

// 从channel队列上获取一个frame，如果队列上为空，则直接在临时对象池之外新建一个Frame。
func (c *channelFramePool) Get() *Frame {
	select {
	case f :=<-c:
		return f
	default:
		return NewFrame(MaxFramePayloadSize)
	}
}

// 当channel队列上满时，则frame只能靠GC了。
func (c *channelFramePool) Release(f *Frame) {
	select {
	case c <- f:
	default:
	}
}
```

# call options

这节内容比较少，我们增加call req和call res协议payload部分的transport header相关内容

如果大家对tchannel的transport headers不了解，可以看看tchannel协议规范中的protocal部分call req和call res的transport headers。

```shell
nh~1 (key~1 value~1){nh}
```

```shell
type Format string

func (f Format) String() string {
	return string(f)
}

const (
	HTTP Format = "http"
	JSON Format = "json"
	Raw Format = "raw"
	Thrift Format = "thrift"
)

// 对于transport headers部分，当key="as" ,也即the Arg Scheme。 value可以为：thirft, sthrift, json, http和raw。这里我们看到没有使用sthrift协议传输

// transport headers所有相关的key
type CallOptions struct {
	// the Arg Scheme, 协议类型
	Format Format
	// 一个请求去指定的节点
	ShardKey string
	// 重试相关Retry Flags
	RequestState *RequestState
	// ...
	RoutingKey string
	// 当协议不指定服务名时，可以通过该参数指定
	RoutingDelegate string
	// 发起请求的调用名
	callerName string
}

// 由上面和tchannel协议对比，我们可以看到，这里还缺少双方通信的Host:Port， sepculative execution，Failure Domain熔断几个key。

// 默认的call options是空值
var defaultCallOptions = &CallOptions{}

// 这个transportHeaders在message章节说过.
//
// 为何这里直接把the Arg Scheme设置为raw协议呢？是默认值为原生？
func (c *CallOptions) setHeaders(headers transportHeaders) {
	// 把call options参数赋值给transport headers
	headers[ArgScheme] = Raw.String()
	c.overrideHeaders(headers)
}

func (c *CallOptions) overrideHeaders(headers transportHeaders) {
	// 把call options赋值给transport headers
	if c.Format != "" {
		headers[ArgScheme] = c.Format.String()
	}
	if c.ShardKey != ""{
		headers[ShardKey] = c.ShardKey
	}
	
	if c.RoutingKey != "" {
		headers[RoutingKey] = c.RoutingKey
	}
	
	if c.RoutingDelegate != ""{
		headers[RoutingDelegate] = c.RoutingDelegate
	}
	
	if c.callerName != "" {
		headers[CallerName] = c.callerName
	}
}

// 只是赋值个ArgScheme， 其他不用改变。用这么大的方法？
func setResponseHeaders(reqHeaders, respHeaders transportHeaders) {
	respHeaders[ArgScheme] = reqHeaders[ArgScheme]
}
```

# 总结

对于transport headers比较奇怪的是，在tchannel协议规范中并不存在routing key，但是实现中却存在。

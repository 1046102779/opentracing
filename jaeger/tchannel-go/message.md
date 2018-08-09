# message interface

针对tchannel协议规范，rpc消息类型包括九种，分别是：

```shell
type message Type byte

const (
	messageTypeInitReq messageType = 0x01
	messageTypeInitRes messageType = 0x02
	messageTypeCallReq messageType = 0x03
	messageTypeCallRes messageType = 0x04
	messageTypeCallReqContinue messageType = 0x13
	messageTypeCallResContinue messageType = 0x14
	messageTypePingReq messageType = 0xd0
	messageTypePingRes messageType = 0xd1
	messageTypeError messageType = 0xFF
)

// message的通用interface
// 每个消息都有一个message id、消息类型、读取和写入数据
type message interface {
	ID() uint32
	
	messageType() messageType
	
	read(r *typed.ReadBuffer) error
	
	write(w *typed.WriteBuffer) error
}
```

**针对以上的9种消息类型，我们分别对rpc传输协议帧的payload部分进行特定解析，协议帧的头部是固定解析的，只是payload部分针对不同消息类型，进行特定解析，这部分依赖于message interface的具体实现，接下来的具体实现是针对tchannel协议规范的payload部分，有些可以聚合在一起解析的，就弄在一起了。**

# message implementations

接下来会有10种实现, 9+1， 前9种就是上面的消息类型；外加一个默认消息空实现。

注意，这里是网络流的数据写入和读取，所以存在大小端。tchannel也是这么规范的

## noBodyMsg

默认消息读取和写入消息为空实现

```shell
type noBodyMsg struct {}

func (noBodyMsg) read(r *typed.ReadBuffer) error { return nil }
func (noBodyMsg) write(w *typed.WriteBuffer) error { return nil }
```

## init msg

消息类型：init req

**payload具体格式：version:2 nh:2(key~2 value~2){nh}**

nh表示header的键值对{host_port: xxx, process_name: xxx, tchannel_language: xxx, ...}

大家可以看看tchannel协议规范，然后再看代码，更容易理解。

init msg是rpc连接初始化握手时的消息

init msg消息是call init req和call init res消息的基类，在tchannel消息头部会有一些固定参数，如下所示:

```shell
type initParams map[string]string

const (
	// 在前文介绍过，这些分别是：remote service HOST:PORT与process_name、实现tchannel协议的语言和语言版本号（如：go, go1.10.1）以及tchannel协议版本号
	InitParamHostPort = "host_port"
	InitParamProcessName = "process_name"
	InitParamTChannelLanguage = "tchannel_language"
	InitParamTChannelLanguageVersion = "tchannel_language_version"
	InitParamTChannelVersion = "tchannel_version
)
```

接下来，就是init msg的具体读写tchannel协议头部实现

```shell
// init msg结构
type initMessage struct {
	id uint32
	Version uint16
	initParams initParams
}

// 读取网络数据流协议payload部分, 协议帧的header部分已被解析了。
func (m *initMessage) read(r *typed.ReadBuffer) error {
	// 根据协议帧，消息类型为init req和init res。
	// payload: version:2, nh:2 (key~2 value~2){nh}
	// nh表示headers
	
	// 根据上面的描述，我们首先获取version信息, 默认是tchannel version为2.
	// 版本占用2字节
	m.Version = r.ReadUint16()
	
	// 初始化headers键值对存储
	m.initParams = initParams{}
	// 再读取键值对数量, 两个字节
	np := r.ReadUint16()
	for i:=0; i < int(np); i++ {
		// header的key:value都是2字节
		k := r.ReadLen16String()
		v := r.ReadLen16String()
		// 存储header键值对到initParams中
		m.initParams[k] = v
	}
	
	return r.Err()
}

// 与read实现类似，把initMessage对象写入到网络数据流中
func (m *initMessage) write(r *typed.ReadBuffer) error {
	// 首先写入version信息, 占用2字节
	w.WriteUint16(m.Version)
	// 写入headers键值对的长度, 长度2字节
	w.WriteUint16(uint16(len(m.initParams)))
	
	// 写入header的key-value
	for k, v := range m.initParams {
		w.WriteLen16String(k)
		w.WriteLen16String(v)
	}
	
	return w.Err()
}

// 获取消息id
func (m *initMessage) ID() uint32 {
	return m.id
}
```

## 消息类型：init req

由于init req和init res的payload部分格式是相同的，在上面init message的write和read基础上只需要添加消息类型就行了。

```shell
type initReq struct {
	initMessage
}

func (m *initReq) messageType() messageType {
	return messageTypeInitReq
}
```

## 消息类型：init res

同上，只需要增加消息类型，就实现了message interface

```shell
type initRes struct {
	initMessage
}

func (m *initRes) messageType() messageType {
	return messageTypeInitRes
}
```

## tranpsort headers

对于消息类型为call req和call res，这些帧的payload部分有transport headers数据。

transport headers意图在传输和路由层控制一些东西，同时又希望代价小。应用层headers在更高层表达，不在这一层。 具体见tchannel协议规范

|name|	req|	res |	description| 中文解释 |
|---|---|---|---|---|
|as	|Y |	Y	|the Arg Scheme| 传输协议：thrift, sthrift, json, http, raw |
|cas|	Y	|N	|Claim At Start| remote service host:port. 当工作开始时发送这个claim消息|
|caf|	Y|	N|	Claim At Finish| remote service host:port. 当response回复时，远端host：port，也就是req的本地host：port|
|cn|	Y|	N|	Caller Name| 调用名 |
|re|	Y|	N|	Retry Flags| 重试策略|
|se|	Y|	N|	Speculative Execution| 节点选择机制 |
|fd|	Y|	Y|	Failure Domain| 熔断机制 |
|sk	|Y|	N|	Shard key| 如果你想某个请求到一个特定节点上，使用sk|
|rd|	Y|	N|	Routing Delegate| 我们也可以通过rd路由到指定服务，而不是需要直接在call req中指定|

有关这两个call req和call res两个协议消息，大家可以参考tchannel协议规范。


下面是transport headers实现细节

```shell
type TransportHeaderName string

func (cn TransportHeaderName) String() string {
	return string(cn)
}

const (
	ArgScheme TransportHeaderName = "as"
	
	CallerName TransportHeaderName = "cn"
	
	ClaimAtFinish TransportHeaderName = "caf"
	
	ClaimAtStart TranpsortHeaderName = "cas"
	
	FailureDomain TransportHeader = "fd"
	
	ShardKey TransportHeaderName = "sk"
	
	RetryFlags TransportHeaderName = "re"
	
	SpeculatvieExecution TranpsortHeaderName = "se"
	
	RoutingDelegate TransportHeaderName = "rd"
	
	RoutingKey TransportHeaderName = "rk"
)

type transportHeaders map[TransportHeaderName]string

// 直接读取payload部分的 nh:1 (key~1 value~1){nh}
// 上面直接读取这部分的原因是，可以为call req和call res共用transport headers
func (ch transportHeaders) read(r *typed.ReadBuffer) {
	// 先读取1字节数据, 获取键值对的长度
	nh := r.ReadSingleByte()
	// 根据键值对的长度，遍历获取并存储到map中
	for i := 0; i < int(hn); i++ {
		k := r.ReadLen8String()
		v := r.ReadLen8String()
		ch[TransportHeaderName(k)] = v
	}
}

// 同上，写入map存储到网络数据流中。
func (ch transportHeaders) write(r *typed.WriteBuffer) {
	// 获取map的键值对长度
	w.WriteSingleByte(byte(len(ch)))
	
	// 键值对写入tchannel协议网络流中
	for k, v:= range ch {
		w.WriteLen8String(k.String())
		w.WriteLen8String(v)
	}
}
```

## 消息类型：call req

消息类型为call req的payload数据读取和写入, 实现了message interface

```shell
call req payload:
flags:1 ttl:4 tracing:25
service~1 nh:1 (hk~1 hv~1){nh}
csumtype:1 (csum:4){0,1} arg1~2 arg2~2 arg3~2
```

由上面的payload部分，我们可以知道callReq的大体结构如下：

```shell
type callReq struct {
	id uint32
	TimeToLive time.Duration
	Tracing Span
	Headers transportHeaders
	Service string
}

// 消息id
func (m *callReq) ID() uint32 {
	return m.id
}

func (m *CallReq) messageType() messageType {
	return messageTypeCallReq
}

// 读取payload部分，有个疑问，1字节的flags没有读取 why? ::TODO
func (m *CallReq) read(r *typed.ReadBuffer) error {
	// 4字节的ttl
	m.TimeToLive = time.Duration(r.ReadUint32())*time.Millisecond
	// 25字节的trace
	m.Tracing.read(w)
	// 1字节的remote service name
	m.Service = w.ReadLen8String()
	// 读取nh:1 (key~1 value~1){nh}
	m.Headers = transportHeaders{}
	m.Headers.read(r)
}

// 写入payload部分网络流，疑问同上，flags:1
func (m *callReq) write(w *typed.WriteBuffer) error {
	// 4字节的ttl
	w.WriteUint32(uint32(m.TimeToLive / time.Millisecond))
	// 25字节的trace
	m.Tracing.Write(w)
	// 1字节的remote service name
	w.WriteLen8String(m.Service)
	// transport headers
	m.Headers.write(w)
}
```

**call req的payload部分首1字节flag并没有在read和write读取和写入，为啥呢？**

## 消息类型：call req continue

由于call req continue和call res continue协议帧的payload部分：

```shell
flags:1 csumtype:1 (csum:4){0,1} {continuation}
```

很简单，直接过滤，所以这两类消息的读取和写入，直接组合nBodyMsg即可。

```shell
type callReqContinue struct {
	noBodyMsg
	id uint32
}

// 返回消息ID
func (c *callReqContinue) ID() uint32 {
	return c.id
}

func (c *callReqContinue) messageType() messageType {
	return messageTypeCallReqContinue
}
```

## 消息类型：call res continue

同上， **只是有疑问的是，为何组合noBodyMsg，而是自己写一个和它相同的实现....?**

```shell
type callResContinue struct {
	id uint32
}

// 返回消息ID
func (c *callResContinue) ID() uint32 {
	return c.id
}

// 返回消息类型
func (c *callResContinue) messageType() messageType {
	return messageTypeCallResContinue
}

func (c *callResContinue) read(r *typed.ReadBuffer) error {
	return nil
}

func (c *callResContinue) write(r *typed.ReadBuffer) error {
	return nil
}
```

## 消息类型：call res

类似call req消息类型的解析：

```shell
call res payload:
flags:1 code:1 tracing:25
nh:1 (hk~1 hv~1){nh}
csumtype:1 (csum:4){0,1} arg1~2 arg2~2 arg3~2
```

其中上面的code：

| code	| name |	description| 中文解释|
| --- | --- |--- | --- |
| 0x00|	OK|	everything is great and we value your contribution.| 响应成功 |
|0x01	|Error|	application error, details are in the args.| 应用错误，非业务逻辑错误码 |

```shell
type callRes struct {
	id uint32
	ResponseCode ResponseCode
	Tracing Span
	Headers transportHeaders
}

// 返回消息ID
func (m *callRes) ID() uint32 {
	return m.id
}

// 返回消息类型
func (m *callRes) messageType() messageType {
	return messageTypeCallRes
}

// 和前面的call req消息类型一样，没有解析首字节flags
func (m *callRes) read(r *typed.ReadBuffer) error {
	// 1字节的响应码
	m.ResponseCode = ResponseCode(r.ReadSingleByte())
	// 25字节的trace
	m.Tracing.read(r)
	// transport headers
	m.Headers = transportHeader{}
	m.Headers.read(r)
	return r.Err()
}

// 同上
func (m *callRes) write(w *typed.ReadBuffer) error {
	// 写入1字节的响应码
	r.WriteSingleByte(byte(m.ResponseCode))
	// 写入25字节的trace
	m.Tracing.write(w)
	// 写入transport headers
	m.Headers.write(w)
	return w.Err()
}
```

## error message

rpc内部系统错误或者tchannel协议错误, 帧payload部分的格式：

```shell
code:1 tracing:25 message~2
```

```shell
type errorMessage struct {
	id uint32
	errCode SystemErrCode
	tracing Span
	message string
}

// 返回消息ID
func (m *errorMessage) ID() uint32 {
	return m.id
}

// 返回消息类型
func (m *errorMessage) messageType() messageType {
	return messageTypeError
}

// 读取错误消息的payload部分
func (m *errorMessage) read(r *typed.ReadBuffer) error {
	// 1字节的错误码
	m.errCode = SystemErrCode(r.ReadSingleByte())
	// 25字节的trace
	m.tracing.read(r)
	// 2字节的错误消息
	m.message = r.ReadLen16String() 
	return r.Err()
}

// 写入错误消息帧的payload部分
func (m *errorMessage) write(w *typed.WriteBuffer) error {
	// 写入1字节的错误码
	w.WriteSingleByte(byte(m.erroCode))
	// 写入25字节的trace
	w.tracing.write(w)
	// 写入16字节的错误消息字符串
	w.WriteLen16String(m.message)
	return w.Err()
}

func (m errorMessage) AsSystemError() error {
	return NewSystemError(m.errCode, m.message)
}

func (m errorMessage) Error() string {
	return m.AsSystemError().Error()
}
```

## 消息类型：ping req

由于ping req和ping res的payload部分为空，所以read和write使用noBodyMsg即可

```shell

type pingReq struct {
	noBodyMsg
	id uint32
}

// 获取消息ID
func (c *pingReq) ID() uint32 {
	c.id
}

// 获取消息类型
func (c *pingRes) messageType() messageType {
	return messageTypePingReq
}
```

## 消息类型：ping res

处理同上

```shell
type pingRes struct {
	noBodyMsg
	id uint32
}

// 获取消息ID
func (c *pingRes) ID() uint32 {
	c.id
}

// 获取消息类型
func (c *pingRes) messageType() messageType {
	return messageTypePingRes
}
```

## 彩蛋

最后一个莫名的方法放在了messages.go尾部

```shell
// 该方法把消息帧中的payload部分的span获取并返回
// 
// 该帧消息类型为call req，可以先查看下call req payload协议格式
// _spanIndex为trace的索引首位置，_spanLength为长度25
// 然后再获取span
func callReqSpan(f *Frame) Span {
    rdr := typed.NewReadBuffer(f.Payload[_spanIndex : _spanIndex+_spanLength])
    var s Span
    s.read(rdr)
    return s
}
```

## 总结

本节针对所有的协议帧消息类型进行了聚合，并实现了message interface接口，针对每个消息协议帧的payload部分进行了write和read操作，具体可以参考tchannel协议规范。

有个疑问，上文也提及了，就是flags占用1字节的问题，并没有在read和write读取和写入。后续再看，::TODO

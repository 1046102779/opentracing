# connection结构声明

```shell
// PeerVersion表示tchannel协议的语言、语言版本和tchannel版本号
// 例如：golang, golang1.10.1， tchannel 0x02
// tchannel协议数据传输时需要填充
type PeerVersion struct {
	Language string `json:"language"`
	LanguageVersion string `json:"languageVersion"`
	TChannelVersion string `json:"tchannelVersion"`
}

// PeerInfo用于表示Peer服务信息
type PeerInfo struct {
	HostPort string `json:"hostPort"`
	ProcessName string `json:"processName"`
	IsEphemeral bool `json:"isEphemeral"`
	Version PeerVersion `json:"version"`
}

// 本地Peer信息，加上了serviceName
type LocalPeerInfo struct {
	PeerInfo
	
	ServiceName string `json:"serviceName"`
}

// connectionOptions 用于控制一个连接的行为
type ConnectionOptions struct {
	// tchannel协议所需要的存储空间
	// 这里用一个临时对象池
	FramePool FramePool
	
	// 发送到连接上的buffer大小
	SendBufferSize int
	
	// tchannel一次传输数据的校验类型
	ChecksumType ChecksumType
	
	// 标记调出的packets
	TosPriority tos.ToS
	
	// 服务的健康检查
	HealthChecks HealthCheckOptions
}

// connectionEvents用于连接上事件的触发
type connectionEvents struct {
	// 当connection state变为active时，会触发该事件
	OnActive func(c *Connection)
	
	// 当connection state 变为closing时，会触发该事件
	OnCloseStateChange func(c *Connection)
	
	// 当message change增加或者移除时，会触发该事件
	OnExchangeUpdated func(c *Connection)
}

// 连接到远程serviceName的一个connection
type Connection struct {
	channelConnectionCommon
	
	// connection ID全局唯一身份
	connID uint32
	
	// 控制connection的行为
	opts ConnectionOpts
	
	// net连接的维护和管理，这个才是真正的发起者，connection只是一层封装
	conn net.Conn
	
	// local service相关信息
	localPeerInfo LocalPeerInfo
	
	// remote service相关信息
	remotePeerInfo PeerInfo
	
	// 发送帧到channel
	sendCh chan *Frame
	// 停止goroutine的信号channel
	stopCh chan struct{}
	// connection state
	state connectionState
	// connection state lock
	statMut sync.RWMutex
	
	// inbound message exchange set
	inbound *messageExchangeSet
	
	// outbound message exchange set
	outbound *messageExchangeSet
	
	// connection接收数据的处理方法
	handler Handler
	
	// next message id
	nextMessageID atomic.Uint32
	
	// connection events
	events connectionEvents
	
	// connection的tags
	commonStatsTags map[string]string
	
	// 后面再看 ::TODO
	relay *Relayer
	
	// outbound host:port
	outboundHP string
	
	closeNetworkCalled atomic.Int32
	stoppedExchanges atomic.Uint32
	pendingMethods atomic.Int64
	remotePeerAddress peerAddressComponents
	
	// 健康检查相关
	healthCheckCtx context.Context
	healthCheckQuit context.CancelFunc
	healthCheckDone chan struct{}
	healthCheckHistory *healthHistory
	
	lastActivity atomic.Int64
}
```

# connection相关行为

connection state有四种，分别是connectionActive(active)、connectionStartClose(开始关闭，且拒绝新来的请求，流出的请求时允许的)、connectionInboundClosed(流入的所有请求都已处理完成，且流出的请求正在完成或者超时中)和connectionClosed(完全关闭)。这四个state是有时间顺序的

```shell
// 计算当前连接传输的数据还有多长生存时间
func getTimeout(ctx context.Context) time.Duartion {
	deadline, ok := ctx.Deadline()
	if !ok {
		return DefaultConnectionTimeout
	}
	
	return deadline.Sub(time.Now())
}

func (co ConnectionOptions) withDefaults() ConnectionOptions {
	// 默认使用crc32校验
	if co.ChecksumType == ChecksumTypeNone {
		co.ChecksumType = ChecksumTypeCrc32
	}
	
	// 默认使用自带的Pool
	if co.FramePool == nil {
		co.FramePool = DefaultFramePool
	}
	
	// 发送到connection的传输数据大小默认为512个字节
	if co.SendBufferSize <= 0 {
		co.SendBufferSize = defaultConnectionBufferSize
	}
	
	// 默认的健康检查机制
	co.HealthChecks = co.HealthChecks.withDefaults()
}

// 我也是醉了，把channel.go的内容放在connection.go文件中
//
// Socket类用4个整数表示服务类型
// 0x02:低成本(二进制的倒数第二位为1)
// 0x04:高可靠性(二进制的倒数第三位为1)
// 0x08:最高吞吐量(二进制的倒数第四位为1)
// 0x10:最小延迟(二进制的倒数第五位为1)
func (ch *Channel) setConnectionTosPriority(tosPriority tos.ToS, c net.Conn) error {
	tcpAddr, isTCP := c.RemoteAddr().(*net.TCPAddr)
	if !isTCP {
		return nil
	}
	
	var err error
	switch ip := tcpAddr.IP {
		... // 为这个传入的连接c，设置服务类型tosPriority
	}
}

func (ch *Channel) newConnection(
	conn net.Conn, 
	initialID uint32, 
	outboundHP string,
	remotePeer PeerInfo,
	remotePeerAddress peerAddressComponents, 
	events connectionEvents) *Connection {
	// 控制connection的行为，默认值
	ops := ch.conectionOptions.withDefaults()
	
	// 全局connection唯一ID
	connID := _nextConnID.Inc()
	... // log records
	
	// 获取channel所在的peerInfo信息
	peerInfo := ch.PeerInfo()
	
	c := &Connection{
		channelConnectionCommon: ch.chnnelConnectionCommon,
		
		connID: connID,
		conn, conn,
		opts: opts, //connection控制行为
		state: connectionActive,
		// 可以channel发送512帧
		sendCh: make(chan *Frame, opts.SendBufferSize),
		stopCh: make(chan struct{}),
		localPeerInfo: peerInfo,
		remotePeerInfo: remotePeer,
		remotePeerAddress: remotePeerAddress,
		outboundHP: outboundHP,
		// 对于message exchange，在mex.go中讲解 ::TODO
		inbound: newMessageExchangeSet(log, messageExchangeSetInbound),
		outbound: newMessageExchangeSet(log, messageExchangeSetOutbound),
		handler: ch.handler,
		events: events,
		// connection tags
		commonStatsTags: ch.mutable.commonStatsTags,
		healthCheckHistory: newHealthHistory(),
		// 上一次活跃时间记录，idle sweep进行定期超时空闲连接的检测和清除
		lastActivity: *atomic.NewInt64(ch.timeNow().UnixNano()),
	}
	
	设置net.Conn的socket请求类型
	ch.setConnectionTosPriority(tosPriority, conn)
	
	// message id
	c.nextMessageID.Store(initialID)
	... // log, inbound, outbound assign
	
	// 这个还不清楚 ::TDO
	if ch.RelayHost() !=nil {
		c.relay = NewRelayer(ch, c)
	}
	
	// 当connection state变为active时，触发事件
	c.callOnActive()
	
	// 启动两个goroutine，分别接受和发送connection上的数据
	go c.readFrames(connID)
	go c.writerFrames(connID)
	return c
}

func (c *Connection) callOnActive() {
	... // log records
	
	// 当触发connection state为active事件时，直接调用OnActive事件
	if f := c.events.OnActive; f != nil {
		f(c)
	}
	
	// 同时，需要开启goroutine，定时对connection进行健康状态检查
	if c.opts.HealthChecks.enabled() {
		c.healthCheckCtx, c.healthCheckQuit = context.WithCancel(context.Background())
		c.healthCheckDone = make(chan struct{})
		go c.healthCheck(c.connID)
	}
}

// 当拒绝新流入的请求，且流出的请求不受限制时的call, 状态2
func (c *Connection) callOnCloseStateChange() {
	if f := c.events.OnCloseStateChange; f != nil {
		f(c)
	}
}

// message exchange增加或者移除时的call
func (c *Connection) callOnExchangeChange() {
	if i := c.events.OnExchangeUpdated; f != nil {
		f(c)
	}
}

// 对connection做健康检查时的ping message
func (c *Connection) ping(ctx context.Context) error {
	 ... // if the connection is closed, return
	 
	 defer c.pendingExchangeMethodDone()
	 // ping 请求，请求类型：PING_REQ 0xd0
	 req := &pingReq{id: c.NextMessageID()}
	 
	 // 创建一个message exchange
	 mex, err := c.outbound.newExchange(ctx, c.opts.FramePool, req.messageType(), req.ID(), 1)
	 // 返回后执行remove message exchange操作
	 defer c.outbound.removeExchange(req.ID())
	 
	 // 发送ping request message
	 c.sendMessage(req)
	 // 接收ping response message
	 return c.recvMessage(ctx, &pingRes{}, mex)
}

// 对外提供ping响应服务
func (c *Connection) handlePingRes(frame *Frame) bool {
	// 这个在mex.go中讲解
	c.outbound.forwardPeerFrame(frame)
	return
}

func (c *Connection) handlePingReq(frame *Frame) {
	c.pendingExchangeMethodAdd() // 处理ping请求message exchange
	
	defer c.pendingExchangeMethodDone() // 删除ping请求message exchange
	
	... // connection必须为active状态, 不然是不能发送出去的
	
	// ping请求处理后，在发送ping response到connection上返回, 
	// 这里的messageID，还是发送请求的那个ID
	pingRes := &pingRes{id: frame.Header.ID}
	c.sendMessage(pingRes)
}

// 从FramePool池中获取一个frame对象, 发送消息到connection上
func (c *Connection) sendMessage(msg message) error {
	frame := c.opts.FramePool.Get()
	frame.write(msg) // 写入message到frame中
	
	// 如果connection中的sendCh buffer满了，则表示无法发送msg到connection中,
	// 直接抛出send connection满的错误。
	select {
	case c.sendCh <- frame:
		return nil
	default:
		return ErrSendBufferFull
	}
}

// 从recvCh队列中阻塞读取一个frame，并把数据写入到message，然后释放frame返回。
// 
// 这个应该是外部的服务循环获取接收remote service发送的frame消息
func (c *Connection) recvMessage(
	ctx context.Context, 
	msg message,
	mex *messageExchange) error {
	// 从recvCh中获取一个frame
	frame, err := mex.recvPeerFrameOfType(msg.messageType())
	// 读取frame数据到msg
	err = frame.read(msg)
	// 释放frame到FramePool中
	c.opts.FramePool.Release(frame)
	return err
}

// 获取这个连接的remote service信息
func (c *Connection) RemotePeerInfo() PeerInfo {
	return c.remotePeerInfo
}

// readFrames和writeFrames是每个connection启动的两个goroutine，用来接收和发送帧数据
// readFrames读取该连接上的所有frame，并分发到对应协议类型的消息处理
func (c *Connection) readFrames(_ uint32) {
	// 创建一个大小为FrameheaderSize的存储
	headerBuf := make([]byte, FrameHeaderSize)
	
	for {
		// 从连接中只读取FrameHeaderSize大小的内容，即头部
		io.ReadFull(c.conn, headerBuf)
		
		
		// 从FramePool获取一个frame对象
		frame := c.opts.FramePool.Get()
		// 根据headBuf相关信息，获取帧的body大小，并填充到frame
		frame.ReadBody(headerBuf, c.conn)
		
		// 更新connection的最后一次活跃时间
		c.updateLastActivity(frame)
		
		var relaseFrame bool
		// 是否要通过relay转给其他具体类型的handler处理消息
		if c.relay == nil {
			releaseFrame = c.handleFrameNoRelay(frame)
		} else {
			releaseFrame = c.handleFrameRelay(frame)
		}
		// 是否需要释放frame到FramePool
		if relaseFrame {
			c.opts.FramePool.Release(frame)
		}
	}
}

从handleFrameRelay和handleFrameNoRelay两个方法可以看出，处理frame的中继不包括PING的请求和响应处理，也即PING请求需要立即执行，不需要转发处理
// 对于其他类型，则直接转给connection的relay处理
func (c *Connection) handleFrameRelay(frame *Frame) bool {
	switch frame.Header.messageType {
		case messageTypeCallReq, 
			messageTypeCallReqContinue, 
			messageTypeCallRes,
			messageTypeCallResContinue,
			messageTypeError:
			c.relay.Relay(frame)
		default:
			return c.handleFrameNoRelay(frame)
	}
}

// 这两者把tchannel协议的几种帧类型，全部包含了。
func (c *Connection) handleFrameNoRelay(frame *Frame) bool {
	releaseFrme := true
	
	switch frame.Header.messageType {
	case messageTypeCallReq:
		// call req类型的帧处理
		releaseFrame = c.handleCallReq(frame)
	case messageTypeCallReqContinue:
		// call req continue类型的帧处理
		releaseFrame = c.handleCallReqContinue(frame)
	case messageTypeCallRes:
		// call res类型的帧处理
		releaseFrame = c.handleCallRes(frame)
	case messageTypeCallResContinue:
		// call res continue类型的帧处理
		releaseFrame = c.handleCallResContinue(frame)
	case messageTypePingReq:
		// ping req类型的处理
		releaseFrame = c.handlePingRes(frame)
	case messageTypePingRes:
		// ping res类型的帧处理
		releaseFrame = c.handlePingRes(frame)
	default:
		... // log records
	}
	
	return releaseFrame
}

// 发送帧到connection上, 它从sendCh队列上获取消息，这个队列上的消息是从sendMessage来的
func (c *Connection) writeFrames(_ uint32) {
	for {
		select {
		case f := <-c.sendCh:
			// 从sendCh队列上获取一个frame, 并更新连接的活跃时间
			c.updateLastActivity(f)
			
			// 发送帧到connection上
			err := f.WriteOut(c.conn)
			// 释放帧到FramePool临时对象池
			c.opts.FramePool.Release(frame)
			
		case <- c.stopCh:
			// 外界信号，goroutine退出
			c.closeNetwork()
			return
		}
	}
}

// 当connection上有帧消息时，连接不空闲。发送和接收帧时需要更新连接的最新数据流时间
// 
// 注意一点：connection的健康检查PING类型帧消息，不属于有效帧数据。空闲连接不针对它
func (c *Connection) updateLastActivity(frame *Frame) {
	switch frame.Header.messageType {
	case messageTypeCallReq,
		messageTypeCallReqContinue,
		messageTypeCallRes,
		messageTypeCallResContinue,
		messageTypeError:
		c.lastActivity.Store(c.timeNow().UnixNano())
	}
}

// ::TODO
func (c *Connection) pendingExchangeMethodAdd() bool {
	return c.pendingMethods.Inc() > 0
}

// ::TODO
func (c *Connection) pendingExchangeMethodDone() {
	return c.pendingMethods.Dec()
}

// closeSendCh方法目的是关闭连接的发送对接sendCh
// 
//  作者在这里做了一个操作，如果pendingMethods不为0， 则需要等待，直到为0，才执行关闭操作
// 这里对pendingMethods不太了解，后续再看 ::TODO
//
// 这里使用到了atomic的CAS操作，防止并发。当老值为0时，才会写为math.MinInt32
func (c *Connection) closeSendCh(connID uint32) {
	for !c.pendingMehtods.CAS(0, math.MinInt32) {
		time.Sleep(time.Millisecond)
	}
	
	close(c.stopCh)
}

// 当message exchange移除，且Close被调用时，该方法被调用
// 后续再看这个方法 ::TODO
func (c *Connection) checkExchanges() {
	... // 
}

// 关闭connection
// 后续再看  ::TODO
func (c *Connection) close(fields ...LogField) error {
	... //
}
```

# 总结

每个connection都是local service与remote service建立的连接。且每个connection都有两个goroutine，用来发送和接收帧。同时通过队列channel来实现goroutine的帧数据传输。并且要注意idle sweep的空闲连接时长，要把健康检查的PING帧除开。

有几个概念不太清楚，后续再回头理解，包括pendingMethods，checkExchange等

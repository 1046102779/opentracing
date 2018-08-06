连接上帧消息的流入和流出，都需要进行帧的消息封装和消息类型对应的消息处理；我们在上一节讲解过connection的recvMessage和sendMessage两个goroutine进行整体数据的获取和发送，然后在分发给outbound和inbound的对应消息类型，进行帧处理。

# connection outbound


```shell
func (c *Connection) beginCall(
	ctx context.Context,
	serviceName, methodName string,
	callOptions *CalllOptions) (*OutboundCall, error) {
	now := c.timeNow()
	
	// 获取connection state；若非active状态，则直接返回错误
	switch state := c.readState(); state {
	case connectionActive:
		break
	case connectionStartClose, connectionInboundClosed, connectionClosed:
		return nil, ErrConnectionClosed
	default:
		reutrn nil, errConnectionUnknownState{"beginCall", state}
	}
	
	timeToLive := getTimeout(ctx) // 简写了片段
	if timeToLive < time.Millisecond {
		return nil, ErrTimeout
	}
	
	if !c.pendingExchangeMethodAdd() {
		return nil, ErrInvalidConnectionState
	}
	defer c.pendingExchangeMethodDone()
	
	// 获取下一个message ID
	requestID := c.NextMessageID()
	
	// 创建一个message exchange
	mex, err := c.outbound.newExchange(
		ctx, 
		c.opts.FramePool, 
		messageTypeCallReq,
		requestID,
		mexChannelBufferSize)
	
	... // 如果connection不为active，则返回，并关闭mex
	
	// 本地服务名称, 并设置在头部
	headers := transportHeaders {
		CallName: c.localPeerInfo.ServiceName,
	}
	callOptions.setHeaders(headers)
	if opts := currentCallOptions(ctx); opts != nil {
		opts.overrideHeaders(headers)
	}
	
	封装一个outcall调出的请求参数
	call := new(OutboundCall)
	call.mex = mex
	call.conn = c
	call.callReq = callReq{
		id: requestID,
		Headers: headers,
		Service: serviceName,
		TimeToLive: timeToLive,
	}
	call.statsReporter = c.statsReporter
	call.createStatsTags(c.commonStatsTags, callOptions, methodName)
	call.log = ...
	
	call.messageForFragment = func(initial bool) message{
		if initial {
			return &call.callReq
		}
		
		return new(callReqContinue)
	}
	
	call.contents = newFragmentingWriter(call.log, call, c.opts.ChecksumType.New())
	
	// 封装outcall的response请求参数
	response := new(OutboundCallResponse)
	response.startAt = now
	response.timeNow = c.timeNow
	response.requestState = callOptions.RequestState
	response.mex = mex
	response.log = ...//
	response.span = c.startOutboundSpan(ctx, serviceName, methodName, call, now)
	response.messageForFragment = .../.
	response.contents = newFragmentReader(response.log, response)
	response.statsReporter = call.statsReporter
	response.commonStatsTags = call.commonStatsTags
	
	call.response = response
	call.writeMethod([]byte(methodName))
	
	return call
}

// 这里把connection放在的outbound中，所以很多地方的代码组织结构都存在一定的问题
// 
// 处理recvMessage goroutine分发的响应请求
// 消息类型：call res
func (c *Connection) handleCallRes(frame *Frame) bool {
	c.outbound.forwardPeerFrame(frame)
	... //
}

// 处理recvMessage goroutine分发的响应多包请求
// 消息类型：call res continue
func (c *Connection) handleCallResContinue(frame *Frame) bool {
	c.outbound.forwardPeerFrame(frame)
	... // 
}

// OutboundCall是一个从channel通过BeginCall获取到的已经封装tchannel协议帧的部分内容的远程调用，它通过ArgWriter2()和ArgWriter3()填充argument内容，并通过ArgReader2()和ArgReader3()读取响应数据中的argument内容。
//
// OutboundCall结构:
type OutboundCall struct {
	// 写request/response消息
	reqResWriter
	
	// 读写通用消息，比如：头部、trace、服务名等
	callReq callReq
	// 读取帧消息中的argument
	response *OutboundCallResponse
	// 数据统计
	statsReporter StatsReporter
	// 调用连接上的标签
	commonStatsTags map[string]string
}

// 返回读取argument的对象
func (call *OutboundCall) Response() *OutboundCallResponse {
	return call.response
}

// OutboundCall创建标签列表
func (call *OutboundCall) createStatsTags(
	connectionTags map[string]string,
	callOptions *CallOptions,
	method string) {
	call.commonStatsTags = map[string]string{
		"target-service": call.callReq.Service,
	}
	for k, v := range connectionTags {
		call.commonStatsTags[k] = v
	}
	
	if callOptions.Format != HTTP {
		call.commonStatsTags["target-endpoint"] = string(method)
	}
}

// 统计该连接调用的次数，并把method写入到tchannel协议中。
// 
// 写入method到call中
func (call *OutboundCall) writeMethod(method []byte) error {
	call.statsReporter.IncCounter("outbound.calls.send", call.commonStatsTags, 1)
	return NewArgWriter(call.arg1Writer()).Write(method)
}

// 从OutboundCall中获取arg2的writer
func (call *OutboundCall) Arg2Writer() (ArgWriter, error) {
	return call.arg2Writer()
}

// 从OutboundCall中获取arg3的writer
func (call *OutboundCall) Arg3Writer() (ArgWriter, error) {
	return call.arg3Writer()
}

// 从OutboundCall获取connection的本地服务信息
func (call *OutboundCall) LocalPeer() LocalPeerInfo {
	return call.conn.localPeerInfo
}

// 从OutboundCall获取connection的远程服务信息
func (call *OutboundCall) RemotePeer() PeerInfo {
	reutrn call.conn.RemotePeerInfo()
}

// 什么也不做
func (call *OutboundCall) doneSending() {}
```

# OutboundCallResponse

```shell
// OutboundCallResponse是对outbound call的响应
type OutboundCallResponse struct {
	// 从请求中读取argument
	reqResReader 
	
	// 获取响应帧并解析的对象
	callRes callRes
	
	requestState *RequestState
	startedAt time.Time
	timeNow func() time.Time
	span opentracing.Span
	statsReporter StatsReporter
	commonStatsTags map[string]string
}

// 格式化响应帧的argument
func (response *OutboundCallResponse) Format() Format {
	return Format(response.callRes.Headers[ArgScheme])
}

// 从OutboundCallResponse中读取帧的argument，这个是读取arg2
//
// 这里注意的是，我们在OutboundCall写入arg2时，是需要先使用writeMethod方法，再通过Arg2Writer方法写入arg2的
// 
// 这里的话，因为没有不需要知道method信息，所以没有读readMethod方法，所以读取比特流时要先读取method，再读取arg2
func (response *OutboundCallResponse) Arg2Reader() (ArgReader, error) {
	var method []byte
	if err := NewArgReader(response.arg1Reader()).Read(&method); err != nil{
		return nil, err
	}
	
	return response.arg2Reader()
}

// 这里就直接使用arg3Reader方法了。
func (response *OutboundCallResponse) Arg3Reader() (ArgReader, error) {
	return response.arg3Reader()
}

// 如果在意tags对结果的影响，这里可能需要加锁操作。
func cloneTags(tags map[string]string) map[string]string {
	newTags := make(map[string]string, len(tags)
	
	for k, v := range tags {
		newTags[k] = v
	}
	return newTags
}

// 在OutboundCall的doneSending方法中什么也没有做
//
// 而doneReading方法关闭的message exchange
// 对于调出calls，最后一个message是读取call的响应
func (response *OutboundCallResponse) doneReading(unexpected error) {
	now := response.timeNow()
	
	... // span finished
	
	... // statsReporter 做一些数据统计,包括：重试次数、调用成功次数、失败次数、延迟时长等
	
	// message exchange关闭操作
	response.mex.shutdown()
}

func validateCall(ctx context.Context,
		serviceName , methodName string,
		callOpts *CallOptions) error {
	if serviceName == ""{
		return ErrNoServiceName
	}
	
	// 方法长度太大
	if len(methodName) > maxMethodSize {
		return ErrMethodTooLarge
	}
	
	// 调用超时，因为可能是请求拆分成多帧了。
	if _, ok := ctx.Deadline(); !ok {
		return ErrTimeoutRequired
	}
	
	return nil
}
```

# connection inbound

connection inboundCall是对调入的请求和响应, 而connection outboundCall是对调出的请求和响应。

```shell
// handleCallReq方法处理调入的请求，注册一个message exchange到接收更多的fragments中, 并分发它到另一个goroutine处理
// 处理类型：call req
func (c *Connection) handleCallReq(frame *Frame) bool {
	now := c.timeNow()
	... // 如果connection非active状态，则直接返回
	
	// 解析可以前期确定的消息内容；例如：header、trace、存活时间、服务名等
	callReq := new(callReq)
	// 因为是处理调入的请求，所以message id不会变化
	callReq.id = frame.Header.ID
	// 从frame获取一个readableFragment初始化的帧
	initialFragment, err := parseInboundFragment(c.opts.FramePool, frame, callReq)
	
	call := new(InboundCall)
	call.conn = c
	// message exchange method stats
	
	mex, err := c.inbound.newExchange(ctx, 
			c.opts.FramePool,
			callReq.messageType(),
			frame.Header.ID, 
			mexChannelBufferSize,
	)
	
	response :=new(InboundCallResponse)
	response.call = call
	response.calledAt = now
	response.timeNow = c.timeNow
	response.span = c.extractInboundSpan(callReq)
	// 获取span，把baggage信息存储到context上下文进行传播
	if response.span != nil {
		mex.ctx = opentracing.ContextWithSpan(mex.ctx, response.span)
	}
	
	response.mex = mex
	response.conn = c
	response.cancel = cancel
	... // response内容封装填充
	
	call.mex = mex
	call.response = response
	... // call内容封装填充
	
	// 所有封装的信息最后存放到call中，并起一个goroutine分发到channel的Handler中处理inboundCall
	go c.dispatchInbound(c.connID, callReq.ID(), call, frame)
	return false
}

// 处理类型：call req continue
// 转发处理
func (c *Connection) handleCallReqContinue(frame *Frame) bool {
	c.inbound.forwardPeerFrame(frame)
	...
}

func (c *Connection) dispatchInbound(_ uint32, _ uint32, call *InboundCall, frame *Frame) {
	 ... // others
	 
	 // 这个Handle是从channel中继承的, 用来处理分发的请求call给上层应用处理
	 c.handler.Handle(call.mex.ctx, call)
}

InboundCall是一个remote service调出的请求
type InboundCall struct {
	reqResReader
	
	conn *Connection
	// 对调入请求的响应写入argument
	response *InboundCallResponse
	// 被调用的服务名
	serviceName string
	// 被调用的方法名
	// 当调入请求进来后，就可以找到对应的处理方法进行业务处理
	method []byte
	methodString string
	
	headers transportHeaders
	statsReporter StatsReporter
	commonStatsTags map[string]string
}

func (call *InboundCall) ServiceName() string {
	reurn call.serviceName
}

func (call *InboundCall) Method() []byte {
	return call.method
}

// 格式化header中的argument
func (call *InboundCall) Format() Format {
	return Format(call.headers[ArgScheme])
}

// 从header中返回caller name
func (call *InboundCall) CallerName() {
	return call.headers[CallerName]
}

// 从header中返回shard key
func (call *ShardKey() string {
	return call.headers[ShardKey]
}

// 从header中返回routing key
func (call *InboundCall) RoutingKey() string {
	return call.headers[RoutingKey]
}

// 从header中返回routing delegate
func (call *InboundCall) RoutingDelegate() string {
	return call.headers[RoutingDelegate]
}

// 返回local service的服务信息
func (call *InboundCall) LocalPeer() LocalPeerInfo {
	return call.conn.localPeerInfo
}

// 返回remote service的服务信息
func (call *InboundCall) RemotePeer() PeerInfo {
	return call.conn.RemotePeerInfo()
}

// 返回InboundCall的参数信息
func (call *InboundCall) CallOptions() *CallOptions {
	return &CallOptions{
		callerName: call.CallerName(),
		Format: call.Format(),
		ShardKey: call.ShardKey(),
		RoutingDelegate: call.RoutingDelegate(),
		RoutingKye: call.RoutingKey(),
	}
}

// 从request stream读取arg1，也即method
// 
// method methodString都是method name，为何要两份，设计...
func (call *InboundCall) readMethod() error {
	var arg1 []byte
	NewArgReader(call.arg1Reader()).Read(&arg1) // 读取arg1
	call.method = arg1
	call.methodString = string(arg1)
	return nil
}

// 从InboundCall获取arg2
func (call *InboundCall) Arg2Reader() (ArgReader, error) {
	return call.arg2Reader()
}

// 从InboundCall获取arg3
func (call *InboundCall) Arg3Reader() (ArgReader, error){
	return call.arg3Reader()
}

// 提供一个可以访问InboundCallResponse的接口服务
func (call *InboundCall) Response() *InboundCallResponse {
	return call.response
}

func (call *InboundCall) doneReading(unexpected error) {}

// 新增标签, 你会发现，tchannel-go中非常多的commonStatsTags都是通过快照往下传递的，这样我觉得tchannel-go会很吃内存。
func (call *InboundCall) createStatsTags(connectionTags map[string]string) {
	call.commonStatsTags = map[string]string{
		"calling-service": call.CallerName(),
	}
	for k, v := range connectionTags {
		call.commonStatsTags[k] = v
	}
}
```

# InboundCallResponse

当InboundCall处理完调入的请求后，再通过InboundCallResponse把响应数据写入到connection。

```shell
// InboundCallResponse结构
type InboundCallResponse struct {
	reqResWriter
	
	call *InboundCall
	cancel context.CancelFunc
	
	calledAt time.Time
	timeNow func() time.Time
	
	headers transportHeaders
	span opentracing.Span
	statsReporter StatsReporter
	commonStatsTags map[string]string
}

// 读取arg2， 先把数据流method读取出来，在继续读取arg2.
func (response *InboundCallResponse) Arg2Writer() (ArgWriter, error) {
	// 读取method
	NewArgWriter(response.arg1Writer()).Write(nil)
	return response.arg2Writer()
}

// 读取arg3
func (response *InboundCallResponse) Arg3Writer() (ArgWriter, error) {
	return response.arg3Writer()
}

// doneSending关闭message exchange
//
// 对于调入请求，最后一个消息是发送call response
func (response *InboundCallResponse) doneSending() {
	now := response.timeNow()
	
	... // span finished
	
	... // 数据统计，包括延时、APP错误，成功/失败次数
	
	response.cancel()
	if response.err == nil {
		response.mex.shutdown()
	}
}
```

# 总结

针对active connection的调入和调出操作，需要对arg1，arg2和arg3进行写入和读取操作；需要注意的是，method作为arg1，在读取arg2时，如果不关心method，则需要在读取arg2时，先丢弃method所占用的数据流。

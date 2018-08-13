# context

context用于rpc上下文传输

```shell
const defaultTimeout = time.Second

type contextKey int

// tchannel提供了两个header:
const (
	contextKeyTChannel contextKey = iota
	contextKeyHeaders
)

// tchannelCtxParams作为context的value值
type tchannelCtxParams struct {
	// 上下文传递，是否开启trace
	tracingDisabled bool
	// 
	hideListeningOnOutbound bool
	// IncomingCall interface
	call IncomingCall
	// call options 在上节介绍过, 主要是transport headers信息参数
	// 我们可以发现call options和retryOptions信息冗余
	options *CallOptions
	// 重试机制
	retryOptions *RetryOptions
	// 连接超时时间
	connectTimeout time.Duration
}

// IncomingCall接口主要是获取Transport Header参数
func IncomingCall interface {
	CallerName() string
	
	ShardKey() string
	RoutingKey() string
	
	RoutingDelegate() string
	
	LocalPeer() LocalPeerInfo
	
	RemotePeer() PeerInfo
	
	CallOptions() *CallOptions
}

// 通过contextKeyTChannel获取上下文context的value值tchannelCtxParams
func getTChannelParams(ctx context.Context) *tchannelCtxParams {
	if params, ok := ctx.Value(contextKeyTChannel).(*tchannelCtxParams); ok {
		return params
	}
	return nil
}

// 通过context builder创建一个context实例
func NewContext(timeout time.Duration) (context.Context, context.CancelFunc) {
	return NewContextBuilder(timeout).Build()
}

// 参数设置可以使用函数slice传递
//
// 通过context builder创建一个context实例
func newIncomingContext(call IncomingCall, timeout time.Duration) (
	context.Context, context.CancelFunc) {
	return NewContextBuilder(timeout).
	setIncomingCall(call).
	Build()
}

// 从context获取tchannel的IncomingCall
func CurrentCall(ctx context.Context) IncomingCall {
	if params := getTChannelParams(ctx); params != nil {
		return params.call
	}
	return call
}

// 从context获取tchannel的call options
func currentCallOptions(ctx context.Context) *CallOptions {
	if params := getTChannelParams(ctx); params != nil{
		return params.call
	}
	
	return nil
}

// 从context获取tchannel中的tracing标记，校验是否开启trace
func isTracingDisabled(ctx context.Context) bool {
	if params := getTChannelParams(ctx); params != nil {
		return params.tracingDisabled
	}
	return nil
}
```

上面主要是构建context上下文或者从上下文获取tchannel的transport header相关参数

## context header

```shell
type ContextWithHeaders interface {
	context.Context
	
	// rpc请求的headers
	Headers() map[string]string
	
	// rpc响应的headers
	ResponseHeaders() map[string]string
	
	// 在context上设置响应headers
	SetResponseHeaders(map[string]string)
	
	// 通过parent context创建子child
	Child() ContextWithHeaders
}


// 这个headerCtx的上下文context的key为contextKeyHeaders
type headerCtx struct {
	context.Context
}

// 在ContextWithHeaders中存在rpc请求头和rpc响应头
type headersContainer struct {
	reqHeaders map[string]string
	respHeaders map[string]string
}

// 通过contextKeyHeaders值获取context的value值headersContainer
func (c headerCtx) headers() *headersContainer {
	if h, ok := c.Value(contextKeyHeaders).(*headersContainer); ok {
		return h
	}
	return nil
}

// 获取rpc请求头
func (c headerCtx) Headers() map[string]string {
	if h := c.headers() ; h != nil {
		return h.reqHeaders
	}
	
	return nil
}

// 获取rpc响应头
func (c headerCtx) ResponseHeaders() map[string]string {
	if h := c.headers(); h!=nil{
		return h.reqHeaders
	}
	
	return nil
}

// 设置rpc响应头部
func (c headerCtx) SetResponseHeaders(headers map[string]string) {
	if h := h.headers(); h != nil {
		h.respHeaders = headers
		return
	}
	
	panic(...)
}

// Child创建一个子context, 且hedersContainer是独立的
func (c headerCtx) Child() ContextWithHeaders {
	var headersCopy headersContainer
	if h := c.headers(); h != nil{
		headerCopy = *h
	}
	
	return Wrap(context.WithValue(c.Context, contextKeyHeaders, &headerCopy))
}

// 通过传入参数context，新建一个ContextWithHeaders
func Wrap(ctx context.Context) ContextWithHeaders {
	hctx := headerCtx{Context: ctx}
	if h := hctx.headers() ; h != nil {
		return hctx
	}
	
	return WrapWithHeaders(ctx, nil)
}

// 把headers以headerContainer的形式存储到context上下文中
func WrapWithHeaders(ctx context.Context, headers map[string]string) ContextWithHeaders {
	h := &headersContainer {
		reqHeaders: headers,
	}
	newCtx := context.WithValue(ctx, contextKeyHeaders, h)
	return headerCtx{Context:newCtx)
}

// 移除context的key为contextKeyTChannel的value值
func WithoutHeaders(ctx context.Context) context.Context {
	return context.WithValue(
		context.WithValue(ctx, contextKeyTChannel, nil),
		contextKeyHeaders,
		nil)
}
```

## context builder

通过ContextBuilder构建一个context，我们在上面已经涉及到contextbuilder了。

```shell
type ContextBuilder struct {
	// 跨进程上下文传递，是否开启trace
	TracingDisabled bool
	
	// 当创建调出的连接时，关闭发送server端的host:port
	hideListeningOnOutbound bool
	
	// true, 强制parentContext被忽略; false, 表示需要合并Parent context
	replaceParentHeaders bool
	
	// 如果值为0， 则ContextBuilder使用默认值设置defaultTimeout;
	Timeout time.Duration
	
	// application headers，json/thrift会把headers编码进arg2
	Headers map[string]string
	
	// call options 前文已经说过; 是transport headers的相关参数列表
	CallOptions *CallOptions
	
	// 重试机制，这个在CallOptions是存在的，为何要独立出来
	RetryOptions *RetryOptions
	
	// 创建一个tchannel连接的timeout
	ConnectTimeout time.Duration
	
	// ParentContext构建一个新的builder，如果ParentConext为空，则直接使用context.Background()
	ParentContext context.Context
	
	// 隐藏字段：不想tchannel外部设置它
	incomingCall IncomingCall
}

// 创建一个ContextBuilder实例
func NewContextBuilder(timeout time.Duration) *ContextBuilder {
	return  &ContextBuilder{
		Timeout: timeout,
	}
}

// 设置ContextBuilder的Timeout参数
func (cb *ContextBuilder) SetTimeout(timeout time.Duration) *ContextBuilder {
	cb.Timeout = timeout
	return cb
}

// 给app添加header
func (cb *ContextBuilder) AddHeader(key, value string) *ContextBuilder {
	if cb.Headers == nil {
		cb.Headers = make(map[string]string)
	}
	cb.Headers[key] = value
	return cb
}

// 给app设置headers
func (cb *ContextHeader) SetHeaders(headers map[string]string) *ContextBuilder {
	cb.Headers = headers
	cb.replaceParentHeaders = true
	return cb
}

// 设置transport headers中的shardkey参数, 通过这个参数可以选择指定的node
func (cb *ContextHeader) SetShardKey(sk string) *ContextBuilder {
	if cb.CallOptions == nil {
		cb.CallOptions = new(CallOptions)
	}
	cb.CallOptions.ShardKey = sk
	return cb
}

// 设置transport headers中的arg scheme
func (cb *ContextHeader) SetFormat(f Format) *ContextBuilder {
	if cb.CallOptions == nil {
		cb.CallOptions = new(CallOptions)
	}
	cb.CallOptions.Format = f
	return cb
}

// 设置transport headers中的routing key。这个在规范协议中没有.
func (cb *ContextBuilder) SetRoutineKey(rk string) *ContextBuilder {
	if cb.CallOptions == nil {
		cb.CallOptions = new(CallOptions)
	}
	cb.CallOptions.RoutingKey = rk
	return cb
}

// 设置transport headers中的routing delegate
func (cb *ContextBuilder) SetRoutineDelegate(rd string) *ContextBuilder {
	if cb.CallOptions == nil {
		cb.CallOptions = new(CallOptions)
	}
	cb.CallOptions.RoutingDelegate = rd
}

// 设置创建连接的超时时间
func (cb *ContextBuilder) SetConnectionTimeout(d time.Duration) *ContextBuilder {
	cb.ConnectTimeout = d
	return cb
}

// 设置屏蔽传输server的host:port
func (cb *ContextBuilder) HideListeningOnOutbound() *ContextBuilder {
	cb.hideListeningOnOutbound = true
	return cb
}

// 关闭trace
func (cb *ContextBuilder) DisableTracing() *ContextBuilder {
	cb.TracingDisabled = true
	return cb
}

// 设置transport headers中的重试机制
func (cb *ContextBuilder) SetRetryOptions(retryOptions *RetryOptions) *ContextBuilder {
	cb.RetryOptions = retryOptions
	return cb
}

// 设置transport headers中的重试机制中的timeoutPerAttempt
func (cb *ContextBuilder) SetTimeoutPerAttempt(timeoutPerAttempt time.Duration) *ContextBuilder {
	if cb.RetryOptions == nil {
		cb.RetryOptions = &RetryOptions{}
	}
	cb.RetryOptions.TimeoutPerAttempt = timeoutPerAttempt
	return cb
}

// 设置context的parentContext
func (cb *ContextBuilder) SetParentContext(ctx context.Context) *ContextBuilder {
	c.ParentContext = ctx
	return cb
}

// 设置InComingCall
func (cb *ContextBuiler) setIncomingCall(call IncomingCall) *ContextBuilder {
	cb.incomingCall = call
	return cb
}

// 获取ContextBuilder的headers。
//
// 注意: 合并
func (cb *ContextBuilder) getHeaders() map[string]string {
	if cb.ParentContext == nil || cb.replaceParentHeaders {
		return cb.Headers
	}
	
	// 获取context key为contextKeyHeaders的value值headersContainer
	parent, ok := cb.ParentContext.Value(contextKeyHeaders).(*headersContainer)
	if !ok || len(parent.reqHeaders) ==0 {
		return cb.Headers
	}
	
	// 合并parent的headers和ContextBuilder的headers
	mergedHeaders := make(map[string]string, len(cb.Headers) + len(parent.reqHeaders)) 
	
	for k, v := range parent.reqHeaders {
		mergedHeaders[k] = v
	}
	
	for k, v := range cb.Headers {
		mergedHeaders[k] = v
	}
	return mergedHeaders
}

// 构建Context实例, context带有key为contextKeyTChanell和contextKeyHeaders。
func (cb *ContextBuilder) Build() (ContextWithHeaders, context.CancelFunc) {
	//	构建context的key为contextKeyTChannel的value值
	params := &tchannelCtxParams {
		options: cb.CallOptions,
		call: cb.incomingCall,
		retryOptions: cb.RetryOptions,
		connectTimeout: cb.ConnectTimeout,
		hideListeningOnOutbound: cb.hideListeningOnOutbound,
		tracingDisabled: cb.TracingDisabled,
	}
	
	// 获取context的key为contextKeyHeaders的value值
	parent := cb.ParentContext
	if parent == nil {
		parent = context.Background()
	} else if headerCtx, ok := parent.(headerCtx); ok {
		parent = headerCtx.Context
	}
	
	var (
		ctx context.Context
		cancel context.CancelFunc
	)
	
	// 继承parent的截止时间或者当前的截止时间
	_, parentHasDeadline := parent.Deadline()
	if cb.Timeout == 0 || parentHasDeadline {
		ctx, cancel = context.WithCancel(parent)
	} else {
		ctx, cancel = context.WithTimeout(parent, cb.Timeout)
	}
	
	ctx = context.WithValue(ctx, contextKeyTChannel, params)
	return WrapWithHeaders(ctx, cb.getHeaders()), cancel
}
```


# 总结

通过context、context headers和context builder，我们可以对rpc跨进程调用的服务，进行参数设置，并传递下去，例如：trace，timeout、retry、headers等

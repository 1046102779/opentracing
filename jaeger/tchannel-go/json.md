接下来的几节，我们都讲述transport headers中的the Arg scheme字段具体实现

首先，讲述json协议的实现

# json context

json传输时使用的context上下文包含：

```shell
key:value={
	contextKeyHeaders: headersContainer,
	contextKeyTChannel: tchannelCtxParams,
}
```

json的Context具体构建实例：

```shell
type Context tchannel.ContextWithHeaders

// 这个带有context的timeout参数设置， 设置contextKeyHeaders和contextKeyTChannel的values值
func NewContext(timeout time.Duration) (Context, context.CancelFunc) {
	ctx, cancel := tchannel.NewContext(timeout)
	return tchannel.WrapWithHeaders(ctx, nil), cancel
}

// 写入contextKeyHeaders
func Wrap(ctx context.Context) Context {
	return tchannel.WrapWithHeaders(ctx, nil)
}

// 对上面Wrap接口的headers参数追加
func WithHeaders(ctx context.Context, headers map[string]string) Context {
	return tchannel.WrapWithHeaders(ctx, headers)
}
```

# json call

```shell
// application错误
type ErrApplication map[string]interface{}

// error interface implementation
func (e ErrApplication) Error() string {
	return fmt.Sprintf("JSON call failed: %v", map[string]interface{}(e))
}

// Client使用json协议调用其他服务
type Client struct {
	ch *tchannel.Channel
	// remote service name
	targetService string
	// rpc client host:port
	hostPort string
}

// client options
func ClientOptions struct{
	HostPort string
}

// 创建一个rpc client
func NewClient(ch *tchannel.Channel, targetService string, opts *ClientOptions) *Client {
	client := &Client{
		ch: ch,
		targetService: targetService,
	}
	if opts!=nil && opts.HostPort != "" {
		client.hostPort = opts.HostPort
	}
	
	return client
}


// makeCall真正发起了异步远程调用, 并阻塞读取response数据
//
// 具体是如何发起的，我们后面会从头梳理一遍整个源码结构
func makeCall(call *tchannel.OutboundCall, 
	headers, arg3In, respHeaders, arg3Out, errorOut interface{}) (
	bool, string, error) {
	if mapHeaders, ok := headers.(map[string]string); ok {
		headers = tchannel.InjectOutboundSpan(call.Response(), mapHeaders)
	}
	
	// 写入arg2参数
	// 说明：调用方法已经在arg1写入了。BeginCall方法中写的
	if err := tchannel.NewArgWriter(call.Arg2Writer()).WriteJSON(headers); err != nil {
		return false, "arg2 write failed", err
	}
	
	// 写入arg3参数，也就是frame最后一个写入值，则协议帧完整可以发送出去了
	//
	// 这个应该是rpc异步调用，真正的触发rpc调用
	if err := tchannel.NewArgWriter(call.Arg3Writer(0).WriteJSON(arg3In); err != nil {
		return false, "arg3 write failed.", err
	}
	
	// 这里阻塞知道该rpc调用响应，并从中读取arg2参数
	if err := tchannel.NewArgReader(call.Response().Arg2Reader()).ReadJSON(respHeaders); err != nil {
		return false, "arg2 read failed.", err
	}
	
	// 读取arg3参数，如果rpc调用application报错，这arg3是报错信息; 否则是remote service的响应业务逻辑数据
	if call.Response().ApplicationError() {
		if err := tchannel.NewArgReader(call.Response().Arg3Reader()).ReadJSON(errorOut); err != nil {
			return false, "arg3 read error failed.", err
		}
	}
	
	if err := tchannel.NewArgReader(call.Response().Arg3Reader()).ReadJSON(arg3Out); err != nil {
		return false, "arg3 read failed", err
	}
	
	return true, "", nil
}

// 开启client的rpc调用, client已经指明了remote service的host:port
// 
// 只需要通过tchannel进行rpc调用
func (c *Client) startCall(ctx context.Context, method string, callOptions *tchannel.CallOptions) (*tchannel.OutboundCall, error) {
	// 若client的hostPort说明了, 则直接使用channel进行method arg1参数封装frame
	if c.hostPort != "" {
		return c.ch.BeginCall(ctx, c.hostPort, c.targetService, method, callOptions)
	}
	
	// 否则，选取一个client与remote service的channel, 进行method arg1 封装frame
	return c.ch.GetSubChannel(c.targetService).BeginCall(ctx, method, callOptions)
}

// 发起一个rpc调用，包含了arg1, arg2, arg3的帧封装，并返回rpc调用的响应数据读取过程
//
// 封装了BeginCall和makeCall两个方法，其中后者发起真正的rpc调用
func (c *Client) Call(ctx Context, method string, arg, resp interface{}) error {
	var (
		headers = ctx.Headers()
		
		respHeaders map[string]string
		respErr ErrApplication
		errAt string
		isOk bool
	)
	
	// 带有重试机制的rpc调用
	err := c.ch.RunWithRetry(ctx, func(ctx context.Context, rs *tchannel.RequestState) error {
		respHeaders, respErr, isOK = nil, nil, false
		errAt = "connect"
		
		// 封装frame帧，以及arg1参数method
		call, err := c.startCAll(ctx, method, &tchannel.CallOptions{
			Format: tchannel.JSON,
			RequestState: rs,
		})
		if err != nil {
			return err
		}
		
		// 封装arg2和arg3, 并真正的rpc异步调用，并阻塞直到response返回
		isOK, errAt, err = makeCall(call, headers, arg, &respHeaders, resp, &respErr)
		return err
	})
	
	if err != nil{
		return fmt.Errorf("%s: %v", errAt, err)
	}
	
	if !isOK {
		return respErr
	}
	
	return nil
}


// 封装makeCall的rpc调用，并设置response headers到context上下文中
func wrapCall(ctx Context, call *tchannel.OutboundCall, method string, arg, resp interface{}) error {
	var (
		respHeaders map[string]string
		respErr ErrApplication
	)
	isOK, errAt, err := makeCall(call, ctx.Headers(), arg, &respHeaders, resp, &respErr)
	if err != nil {
		return fmt.Errorf("%s: %v", errAt, err)
	}
	
	if !isOK {
		return respErr
	}
	
	ctx.SetResponseHeaders(respHeaders)
	return nil
}

// 也是一个封装json frame的rpc调用
func CallPeer(ctx Context, peer *tchannel.Peer, serviceName, method string, arg, resp interface{}) error {
	// 封装底层frame，并写入arg1参数method
	call, err := peer.BeginCall(ctx, serviceName, method, &tchannel.CallOptions{Format: tchannel.JSON})
	if err != nil {
		return err
	}
	
	// 封装arg2、arg3，并发起rpc调用，并返回response，和业务逻辑数据
	return wrapCall(ctx, call, method, arg, resp)
} 

// 使用subchannel进行rpc调用, 其他同CallPeer
func CallSC(ctx Context, sc *tchannel.SubChannel, method string, arg, resp interface{}) error {
	call, err := sc.BeginCall(ctx, method, &tchannel.CallOptions{Format: tchannel.JSON})
	if err != nil{
		return err
	}
	
	return wrapCall(ctx, call, method, arg, resp)
}
```


# json handler

json frame的消息帧接收的处理方法与处理业务逻辑映射

```shell

var(
	// 获取error空指针值的返回去指针值
	typeOfError = reflect.TypeOf((*error)(nil)).Elem()
	// 获取Context空指针值的返回去指针值
	typeOfContext = reflect.TypeOf((*Context)(nil)).Elem()
)

type Handlers map[string]interface{}

// 验证handler的输入和输出类型是否满足要求
func verifyHandler(t reflect.Type) error {
	// handler的输入和输入，都必须为两个参数
	// 其中输入参数分别是: context和业务逻辑的入参struct
	// 输出参数分别是：处理完后的业务逻辑数据和rpc内部错误
	if t.NumIn() != 2 || t.NumOut() != 2 {
		return fmt.Errorf("
	}
	
	// 函数变量，校验入参为指针，且是struct指针，则返回true
	isStructPtr := func(t reflect.Type) bool {
		return t.Kind() == reflect.Ptr && t.Elem().Kind() == reflect.Struct
	}
	
	// 函数变量，校验入参为map，且key为string，则返回true
	isMap := func(t reflect.Type) bool {
		return t.Kind() == reflect.Map && t.Key().Kind() == reflect.String
	}
	
	// 校验入参是否满足上面的两个函数变量任意一个
	validateArgRes := func(t reflect.Type, name string) error {
		if !isStructPtr(t) && !isMap(t) {
			return fmt.Errorf("%v should be a pointer to a struct, or a map[string]interface{}", name)
		}
		return nil
	}
	
	// 校验handler函数的第一个参数是否为context， 这个context是json.Context
	if t.In(0) != typeOfContext {
		return fmt.Errorf("arg0 should be type json.Context")
	}
	
	// 校验handler函数的第二个参数是否为指针struct或者map[string]interface{}
	//
	// 可以被json序列化和返序列化
	if err := validateArgRes(t.In(1), "second argument"); err != nil {
		return err
	}
	
	// 校验handler函数的返回第一个参数是否为指针struct，或者map[string]interface{}
	//
	// 可以被json序列化或者反序列化
	if err := validateArgRes(t.Out(0), "first return value"); err != nil {
		return err
	}
	
	// 校验handler函数的返回第二个参数是否为error类型
	if !t.Out(1).AssignableTo(typeOfError) {
		return fmt.Errorf("second return value should be error")
	}
	
	return nil
}

// handler用于json的rpc调用处理函数统一模板，类似于grpc call函数原型
// grpc: func(ctx context.Context, in *interface{}, out *interface{}) error
//
// 解析handler函数到handler变量值中
type handler struct {
	handler reflect.Value
	argType reflect.Type
	isArgMap bool
	tracer func() opentracing.Tracer
}

// 解析入参f函数值到handler中
func toHandler(f interface{}) (*handler, error) {
	hV := reflect.ValueOf(f)
	// 校验入参函数值的入参和出参是否满足json handler
	if err := verifyHandler(hV.Type()); err != nil{
		return nil, err
	}
	
	// 函数值业务逻辑数据参数的类型赋值给handler的argType
	argType := hV.Type().In(1)
	
	// 返回handler变量值
	return &handler{
		handler: hV, 
		argType: argType, 
		isArgMap: argType.Kind() == reflect.Map,
	}, nil
}

func Register(registrar tchannel.Registrar, funcs Handlers, onError func(context.Context, error)) error {
	handlers := make(map[string]*handler
	
	handler := tchannel.HandlerFunc(
		func(ctx context.Context, call *tchannel.InboundCall) {
			// 找到method对应的handler，然后通过下面的h.Handle方法解析frame，并通过反射拼装业务逻辑的handler闭包函数值，进行调用处理
			h, ok := handlers[string(call.Method())]
			if !ok {
				onError(ctx, fmt.Errorf("call for unregiterred method: %s", call.Method()))
			}
			
			if err := h.Handle(ctx, call); err != nil{
				onError(ctx, err)
			}
		},
	)
	
	func m, f := range funcs {
		// 把服务提供的所有服务方法注册到tchannel中
		//
		// 它是通过registrar.Register注册的, 当进行remote service rpc调用时，通过method方法找到handler，然后再通过上面的tchannel.HandlerFunc进行自下而上的业务函数调用
		h, err := toHandler(f)
		if err != nil {
			return fmt.Errorf("%v can't be used as a handler: %v", m, err)
		}
		
		h.tracer = func() opentracing.Tracer {
			return tchannel.TracerFromRegister(registrar)
		}
		handlers[m] = h
		registrar.Register(handler, m)
	}
	
	return nil
}

// 通过method在tchannel中找到注册的handler后，再通过该Handle方法解析frame，然后通过reflect进行业务逻辑函数拼装，进行调用，并返回给rpc client
func (h *handler) Handle(tctx context.Context, call *tchannel.InboundCall) error {
	var headers map[string]string
	// 这个已经是remote service处理了，从arg2读取headers
	if err := tchannel.NewArgReader(call.Arg2Reader()).ReadJSON(&headers); err != nil {
		return fmt.Errorf("arg2 read failed: %v", err)
	}
	// 抽取context.Context
	tctx = tchannel.ExtractInboundSpan(tctx, call, headers, h.tracer())
	// 从context中获取key为contextKeyHeaders的value值, 返回到ctx中
	ctx := WithHeaders(tctx, headers)
	
	// 解析frame arg3参数
	var (
		arg3 reflect.Value
		callArg reflect.Value
	)
	
	if h.isArgMap {
		// 如果rpc service端的处理函数handler第二个参数是map类型
		//
		// 则通过reflect创建一个map类型
		arg3 = reflect.New(h.argType)
		callArg = arg3.Elem()
	} else {
		// 否则，则是struct指针
		//
		// 创建一个struct
		arg3 = reflect.New(h.argType.Elem())
		callArg = arg3
	}
	
	// 读取frame arg3参数，并写入到arg3中
	// 注意：arg3是rpc client的业务逻辑数据
	if err := tchannel.NewArgReader(call.Arg3Reader(0).ReadJSON(arg3.Interface()); err != nil{
		return fmt.Errorf("arg3 read failed: %v", err)
	}
	
	// 下面是拼装remote service rpc handle函数值
	
	// ctx是上面通过contextKeyHeaders获取的context, 它作为remote service rpc handler的第一个参数，callArg作为第二个参数
	args := []reflect.Value(reflect.ValueOf(ctx), callArg}
	// 通过反射进行函数调用, 这个是业务逻辑处理
	results := h.handler.Call(args)
	
	// results[0]是remote service rpc handler业务逻辑处理完后返回的第一个参数
	// results[1]... 返回的第二个参数error
	res := results[0].Interface()
	err := results[1].Interface()
	
	if err != nil{
		// rpc系统内部错误
		if serr, ok := err.(tchannel.SystemError); ok {
			return call.Response().SendSystemError(serr)
		}
		
		// 应用级别错误
		call.Response().SetApplicationError()
		res = struct{
			Type string `json:"type"`
			Message string `json:"message"`
		}{
			Type: "error",
			Message: err.(error).Error(),
		}
	}
	
	// 写入返回给rpc client的headers
	if err := tchannel.NewArgWriter(call.Response().Arg2Writer()).WriteJSON(ctx.ResponseHeaders()); err != nil {
		return err
	}
	
	// 写入返回给rpc client的业务逻辑数据，并写入到frame arg3
	return tchannel.NewArgWriter(call.Response().Arg3Writer()).WriteJSON(res)
}
```

## 小结

通过json frame协议，我们可以知道tchannel底层不涉及具体协议，只针对frame协议解析，具体的协议json、thrift、http等，是通过json、thrift、http目录下的call.go, handlers.go, context.go处理的，它们是针对具体的协议进行业务逻辑handler注册、解析，和协议参数写入/读取，并通过reflect进行handler函数值的拼装调用, 比较经典，大家可以仔细拜读写源码

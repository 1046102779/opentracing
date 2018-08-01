# channel

有关channel的error，channel-go提供了四种错误类型：

1. `errAlreadyListening`。channel已经在监听了；
2. `errInvalidStateForOp`。对于那个方法，channel处于一个无效状态；
3. `errMaxIdleTimeNotSet`。IdleCheckInterval设置了，但是MaxIdleTime设置为0；
4. `ErrNoServiceName`。没有服务提供；

在channel参数列表中，有一些参数目前还不稳定，未来还可能会变化。

```shell
type ChannelOptions struct {
	// channel connection相关参数
	DefaultConnectionOptions ConnectionOptions
	
	// channel状态变化时，触发回调。它是一个可选参数
	OnPeerStatusChanged func(*Peer)
	
	// 日志输出
	Logger Logger
	
	// relaying相关服务参数， 相关API不稳定
	RelayHost
	
	// 这个channel能够处理的服务名称列表
	RelayLocalHandlers []string
	
	设置relayed调用的最大允许超时时长
	RelayMaxTimeout time.Duration
	
	// RelayTimerVerification会禁用relay定时器池，用于验证一旦他们释放了，这些定时器将不再被使用
	RelayTimerVerification bool
	
	// 这个Reporter用于报告此channel的统计数据
	StatsReporter StatsReporter
	
	// 在单元测试中TimeNow用于覆盖当前时间
	TimeNow func() time.Time
	
	// 在单元测试中，TimeTicket用于覆盖当前time.Ticker事件。
	// 这个API不稳定，将来可能会变化
	TimeTicker func(d time.Duration) *time.Ticker
	
	// MaxIdleTime表示一个空闲连接允许最大空闲时长
	MaxIdleTime time.Duration
	
	// IdleCheckInterval，这个channel区定时扫描所有活跃的连接，检测它们是否已经dropped了。例如，空闲连接超时，则应该被dropped。如果这个值设置为0，则空闲检查被禁止掉；
	IdleCheckInterval time.Duration
	
	Tracer opentracing.Tracer
	
	// 对所有流入到channel的请求进行处理。并覆盖委托给subchannel的默认处理handler
	Handler handler
}

type ChannelState int

const (
	// channel用于client
	ChannelClient ChannelState = iota + 1
	
	// 服务端监听新连接的channel
	ChannelListening
	
	// 接收一个关闭请求的channel。这个channel不再监听，并且新到来的所有连接都被拒绝掉
	ChannelStartClose
	
	// 一个会丢弃所有流入的连接的channel，但是可以有流出的连接。所有流入的调用和新流出的调用都被拒绝
	ChannelInboundClosed
	
	一个已经完全关闭的channel
	ChannelClosed
)

// Channel是一个路由网络的双向流。应用程序可以使用一个Channel通过BeginCall方法发起远程服务调用，或者监听所有流入的调用。想要接收外部请求的应用程序需要使用Serve或者ListenAndServe方法，来监听外部请求。
// ::TODO: 当channel关闭后，Shutdown所有的subchannels
type Channel struct { 	// 该参数是普通对象列表，用于从channel到connection的直接拷贝
	channelConnectionCommon
	
	// channel id标识
	chID uint32
	
	// ::TODO 这个Hyperbahn stack相关概念目前不理解，后续再看
	createdStack string
	
	// 有关channel相关统计tag
	commonStatsTags map[string]string
	
	// 该参数用于控制连接相关行为
	connectonOptions ConnectionOptions
	
	// 维护与该channel对接的所有peers列表
	peers *PeerList
	
	// relay相关服务配置
	relayHost RelayHost
	
	// 在ChannelOptions中解释过
	relayMaxTimeout time.Duration
	
	// 在ChannelOptions中解释过
	relayTimerVerify bool
	
	// 在ChannelOptions中解释过
	handler Handler
	
	// 在ChannelOptions解释过
	onPeerStatusChanged func(*Peer)
	
	// 在Channel中所有存在并发互斥的成员变量
	mutable struct {
		sync.RWMutex
		state ChannelState
		peerInfo LocalPeerInfo
		l net.Listener
		idleSweep *idleSweep
		conns map[uint32]*Connection
	}
}

func NewChannel(serviceName string, optFns ...func(*ChannelOptions)) (*Channel, error) {
	// 有关friendly function api design参考：
	// https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis
	
	// 这里还有添加一个defaultChannelFn的原因在于：
	// 一般情况下是不应该有这个的，但是历史原因有些是传递整个ChannelOptions，则存在覆盖问题。
	optFns = append(optFns, defaultChannelFn)
	for _, optFn := range optFns {
		optFn(opts)
	}
	
	对这个new channel设置一个channel id
	chId := _nextChID.Inc()
	
	// 做一个空闲连接时长参数检查:
	// 如果没有设置连接允许最大空闲时长，但是又设置了空闲连接检查。这不合理，返回errMaxIdleTimeNotSet错误
	if err := opts.validateIdleCheck(); err !=nil {
		return nil, err
	}
	
	// new channel
	ch := &Channel{
		// 从channel流到connection的相关参数
		channelConnectionCommon: channelConnectionCommon {
			// channel可以处理的服务名列表，hash化。 map[string]struct{}
			relayLocal: toStringSet(opts.RelayLocalHandlers),
			// subChannel, 服务所映射的SubChannel， 并发互斥
			subChannels: &subChannelMap{}, 
			tracer, opts.Tracer,
			... // others
		},
		chID: chId,
		// connectionOptions控制connection的行为，包括：
		// checksum类型、协议帧临时对象池、每次读取和写入channel的大小
		// 和健康检查相关参数：健康检查超过设置的秒数则不健康；达到连续健康检查失败次数，则关闭该连接
		connectionOptions: opts.DefaultConnectionOptions.withDefaults(),
		// 选择指定relaying的host:port
		relayHost: opts.RelayHost,
		... // others
	}
	// the root peer list用于连接peers和在subchannels中分享peers, 
	// newChild返回一个新的isolated peer列表，该列表与root peer 列表共享底层peers。
	// ::TODO 具体后面再看
	ch.peers  =newRootPeerList(ch, opts.OnPeerStatusChanged).newChild()
	
	// handler。如果该ChannelOptions设置了handler，则直接作为该channel的handler
	// 否则，则进行callservice的该channel的subchannel分发处理
	// 注意：如果没有在subchannel找到对应服务的channel处理方法，则直接把该channel注册为handler，最终还是该channel处理这个被调服务的请求
	if opts.Handler != nil {
		ch.handler = opts.Handler
	} else {
		ch.handler = channelHandler{ch}
	}
	
	// channel client, NewServerChannel和NewClientChannel都是使用NewChannel方法，则全部是ChannelClient？ ::TODO 后面再看
	ch.mutable.state = ChannelClient 
	// ::TODO 后面再看
	ch.mutable.conns = make(map[uint32]*Connection)
	// 一些通用tag保存，包括: processName, ServiceName, Host
	ch.createCommonStats()
	
	// 为ch的handler封装一些解析帧协议和统计数据的handler。
	if opts.Handler ==nil {
		ch.registerInternal()
	}
	
	// 设置这些Relay接收channel流入进来的请求调用
	if opts.RelayHost != nil {
		opts.RelayHost.SetChannel(ch)
	}
	
	// 启动定时扫描idleConnections的sweep goroutine服务。并关闭超时空闲连接
	ch.mutable.idleSweep = startIdleSweep(ch, opts)
	
	return ch, nil
}

func (ch *Channel) Serve(l net.Listener) error {
	mutable := &ch.mutable
	mutable.Lock()
	defer mutable.Unlock()
	
	// 防止tcp监听关闭后，还有新的连接进来
	// 参见：https://gocn.vip/article/912
	mutable.l = tnet.Wrap(l)
	
	// 改为server listening
	mutable.state = ChannelListening
	... // others
	
	go ch.serve()  // 启动goroutine服务，处理rpc请求
	return nil
}

func (ch *Channel) serve() {
	// 当accept发生错误时，则以2的指数增长，进行延迟接收新的请求；延时最大值1秒
	acceptBackoff := 0 *time.Millisecond
	
	for {
		netConn, err := ch.mutable.l.Accept()
		... // 错误时的延迟处理
		
		acceptBackoff = 0
		
		go func(){
			events := connectionEvents{
				OnActive: ch.inboundConnectionActive,
				OnCloseStateChange: ch.connectionCloseStateChange,
				OnExchangeUpdated: ch.exchangeUpdated,
			}
			if _, err := ch.inboundHandshake(context.Background(), netConn, events); err!=nil{
				netConn.Close()
			}
		}()
	}
}

// ping类似于http ping, 用于校验远程服务的连通性。
// 该channel作为client，进行rpc调用。
func (ch *Channel) Ping(ctx context.Context, hostPort string) error {
	// 找到hostPort与该channel已建立连接
	peer : = ch.RootPeers().GetOrAdd(hostPort)
	
	conn, err := peer.GetConnection(ctx)
	if err != nil {
		return err
	}
	
	return conn.ping(ctx)
}

// 这里是自己添加的map防止conncurrent read、write操作
// 获取该channel的相关统计tag
func (ch *Channel) StatsTags() map[string]string {
	m := make(map[string]string)
	ch.mutable.Lock()
	for k, v := range ch.commonStatsTags {
		m[k] = v
	}
	ch.mutable.Unlock()
}

// 返回该channel也作为peer，对外提供服务的注册服务名
func(ch *Channel) ServiceName() string {
	return ch.PeerInfo().ServiceName
}

// Connect方法表示rpc连接远端server，并返回connectionjjjjj
func (ch *Channel) Connect(ctx context.Context, hostPort string) (*Connection, error) {
	// 只针对client和server处理
	switch state := ch.State(0); state{
		case ChannelClient, ChannelListening:
			break
		... // others
	}
	
	// 尝试从context中获取参数数据，如果是帧拆分，且是子请求。则需要获取ctx数据，设置超时时间
	if params := getTChannelParams(ctx); params !=nil && params.connectTimeout >0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, params.connectTimeout)
		defer cancel
	}
	
	// 通过deadline与now，计算超时剩余时间
	timeout := getTimeout(ctx)
	
	// 该channel作为client，请求和hostPort建立连接
	tcpConn, err := dialContext(ctx, hostPort)
	... // others
	
	// 与channel.serve方法所使用的inboundHandshake类似，后续介绍
	conn, err := ch.outboundHandshake(ctx, tcpConn, hostPort, events)
	if conn != nil {
		... // 其他考量
	}
	return conn, err
}

func (ch *Channel) exchangeUpdated(c *Connection) {
	// 如果连接的host-port为空，表示无效连接
	if c.remotePeerInfo.HostPort == "" {
		return
	}
	// 如果传入的connection，在该channel中找不到host-port已建立的连接，则直接返回
	p, ok := ch.RootPeers().Get(c.remotePeerInfo.HostPort)
	if !ok {
		return
	}
	
	// 找到则更新host-port老连接, 在此方法中会有修改连接的触发动作，修改该host-port的score
	// 以及更新完成时的完成动作
	ch.updatePeer(p)
}

// 增加与该channel的连接
func (ch *Channel) addConnection(c *Connection, direction connectionDirection) bool {
	... // 该操作并发互斥
	
	// 保证当前的server是处理active状态
	if c.readState() != connectionActive {
		return false
	}
	
	... // others
	
	// 添加connection到channel的conns连接池中
	ch.mutable.conns[c.connID] = c
}

func (ch *Channel) connectionActive(c *Connection, direction connectionDirection) {
	// 添加connection到ch对外的conns连接池中
	// 其中这个direction没有使用
	if added := ch.addConnection(c, direction); !added {
		... // closing channel error
		return
	}
	
	// 添加connection到channel的peersbyhostport map中
	ch.addConnectionToPeer(c.remotePeerInfo.HostPort, c, direction)
}

func (ch *Channel) addConnectionToPeer(hostPort string, c *Connection, direction connectionDirection) {
	// 如果已存在hostPort与peers的映射，则只需要增加连接
	// 否则，创建一个peer，并与hostPort建立映射
	p := ch.RootPeers().GetOrAdd(hostPort)
	// 根据调用方向的流入或者流出， 增加peer正确方向的流入或者流出连接到conns列表中
	// 同时对peer增加connection操作，增加一个事件通知
	if err := p.addConnection(c, direction) ; err !=nil {
		..// others
	}
	
	// 更新score和其他事件
	ch.updatePeer(p)
}

// 新增流入的connection：
// 1. 增加channel的conns
// 2. 增加channel的hostPort与peer的映射
// 3. 增加peer的流入conns列表
// 4. 相关新增、更新和完成动作的事件触发
func (ch *Channel) inboundConnectionActive(c *Connection) {
	ch.connectionActive(c, inbound)
}

// 新增流出的connection：
// 1. 增加channel的conns
// 2. 增加channel的hostPort与peer的映射
// 3. 增加peer的流出conns列表
// 4. 相关事件动作的触发
func (ch *Channel) outboundConnectionActive(c *Connection) {
	ch.connectionActive(c, outbound)
}

// 移除已关闭的连接：删除channel.conns中ID为c.connID的连接
func (ch *Channel) removeClosedConn(c *Connection) {
	if c.readState() != connectionClosed {
		return
	}
	
	ch.mutable.Lock()
	delete(ch.mutable.conns, c.connID)
	ch.mutable.Unlock()
}

// 获取channel中连接列表中的最小状态
func (ch *Channel) getMinConnectionState() connectionState {
	minState := connectionClosed
	for _, c:= range ch.mutable.conns {
		if s := c.readState(); s < minState {
			minState = s
		}
	}
	return minState
}

func (ch *Channel) connectionCloseStateChange(c *Connection) {
	// 从channel conns中删除对应的connection
	ch.removeClosedConn(c)
	if peer, ok := ch.RootPeers().Get(c.remotePeerInfo.HostPort); ok {
		// 如果能在channel的peersByHostPort中能够找到, 则需要对peer进行连接closed状态进行相应操作
		peer.connectionCloseStateChange(c)
		ch.updatePeer(peer)
	}
	if c.outboundHP != "" && c.outboundHP != c.remotePeerInfo.HostPort {
		return
	}
	
	ch.mutable.RLock()
	minState := ch.getMinConnectionState()
	ch.mutable.RUnlock()
	
	var updateTo ChannelState
	if minState >= connectionClosed {
		// conns全部处于关闭状态, 则channel也可以关闭了
		updateTo = ChannelClosed
	} else if minState >= connectionInboundClosed && chState == ChannelStartClose {
		// 当channel中的全部处于inbound closed状态时，channel关闭inbound 
		updateTo = ChannelInboundClosed
	}
	
	var updatedToState ChannelState
	if updateTo >0 {
		ch.mutable.Lock()
		if ch.mutable.state == chState {
			ch.mutable.state = updateTo
			updateToState = updateTo
		}
		ch.mutable.Unlock()
		chState = updateTo
	}
	
	// 当channel为closed状态时，则需要从全局MapChannel变量中删除该channel。它已经无法参与rpc了
	if updateToState == ChannelClosed {
		ch.onClosed()
	}
}

// 从全局channelMap中删除该channel
func (ch *Channel) onClosed() {
	removeClosedChannel(ch)
	... // log: Channel closed.
}

// 校验channel状态是否为Closed状态
func (ch *Channel) Closed() bool {
	return ch.State() == ChannelClosed
}

func (ch *Channel) State() ChannelState {
	ch.mutable.RLock()
	state := ch.mutable.state
	ch.mutable.RUnlock()
	
	return state
}

// 优雅地关闭channel
// 这里有个疑问，后续再解答：如果channel的conns数量不为0，则channel的状态为ChannelStartClose状态，后续close流程如何启动，可能是通过动作事件.
func (ch *Channel) Close() {
	ch.mutable.Lock()
	if ch.mutable.l != nil {
		// 优雅地关闭listener，这里有enet.Listener封装，防止关闭监听后，还能接收到新来的connection
		ch.mutable.l.Close() 
	}
	
	// 停止sweep goroutine继续对connections进行空闲时长检测
	ch.mutable.idleSweep.Stop()
	
	ch.mutable.state = ChannelStartClose // channel开始关闭操作流程
	// 如果channel上的连接数为0，则无需进行其他操作
	if len(ch.mutable.conns) == 0 {
		ch.mutable.state = ChannelClosed
		channelClosed = true
	}
	
	for _, c := range ch.mutable.conns {
		connections = append(connections, c)
	}
	ch.mutable.Unlock()
	
	for _, c := range connections {
		c.close(...) // 连接关闭操作
	}
	
	// 当channel的状态处于关闭状态时，则进行close操作, 移除conns
	if channelClosed {
		ch.onClosed()
	}
}

// channel的中继server配置
func (ch *Channel) RelayHost() RelayHost {
	return ch.relayHost
}
```

## channelConnectionCommon结构

该结构用于数据的上下文传播:

```shell
type ChannelConnectionCommon struct {
	// 日志输出
	log Logger
	// ...
	relayLocal map[string]struct{}
	// 连接上的数据统计Reporter
	statsReporter StateReporter
	// tracer
	tracer opentracing.Tracer
	// channel映射到多个子channel
	subChannels *subChannelMap
	timeNow func() time.Time
	timeTicker func(time.Duration) *time.Ticker
}

// 返回全局tracer
func (ccc ChannelConnectionCommon) Tracer() opentracing.Tracer {
	if ccc.tracer !=nil {
		return ccc.tracer
	}
	
	return opentracing.GlobalTracer()
}
```


## 小结

tchannel-go还有很多可以优化的代码，但是目前该项目已经停止继续开发了。所以很多PRs不会合并了

同时为了保证兼容性，有些PRs也不会合并。

例如：[NewChannel and Concurrent map](https://github.com/uber/tchannel-go/pull/710)

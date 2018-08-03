# 全局channel

tchannel-go是一个rpc服务框架，使用所有内部服务和连接都是全局管理，这个依赖于channelMap全局变量。

```shell
// rpc服务治理所有的服务以及相关连接, 一个服务可以有多个channel
var channelMap = struct {
	sync.Mutex
	existing map[string][]*Channel
}{
	existing: make(map[string][]*Channel),
}

// 注册channel到channelMap中
func registerNewChannel(ch *Channel) {
	// 获取该channel的服务名
	serviceName := ch.ServiceName()
	// 创建最大为10M的stack空间
	ch.createdStack = string(getStacks(false))
	... // log records
	
	// 全局锁，增加服务的channel
	channel.Lock()
	defer channelMap.Unlock()
	existing := channelMap.existing[serviceName]
	channelMap.existing[serviceName] = append(existing, ch)
}

// 从channelMap中移除channel
// 
// 交换第i个和最后一个交换
func removeClosedChannel(ch *Channel) {
	channelMap.Lock()
	defer channelMap.Unlock()
	
	// 获取指定服务的channels列表，并找出指定的channel删除
	channels := channelMap.existing[ch.ServiceName()]
	for i, v := range channels {
		if v != ch {
			continue
		}
		channels[i] = channels[len(channels)-1]
		
		channels = channels[:len(channels)-1]
		break
	}
	
	channelMap.existing[ch.ServiceName()] = channels
	return
}
```

以上就是tchannel-go rpc服务治理框架对所有服务的全局管理, 理论上通过channelMap可以知道任何服务的任何细节


# subchannel

subchannels作为channel的属性列表，用于在一个channel允许调用一个指定服务

```shell
type SubChannelOption func(*SubChannel)

// 为subchannel创建一个独立的PeerList， 因为PeerList的PeerScore默认score计算方式：PreferIncomingCalculator。但是想使用LeastPendingCalculator策略。上一节详细介绍
func Isolated(s *SubChannel) {
	s.Lock()
	s.peers = s.topChannel.peers.NewSibling()
	s.peers.SetStrategy(newLeastPendingCalculator())
	s.Unlock()
}

// SubChannel相关属性
type SubChannel struct {
	sync.RWMutex 
	// SubChannel调用所指向的服务名
	serviceName string
	// 所在的channel
	topChannel *Channel
	// rpc调用协议参数
	defaultCallOptions *CallOptions
	// 调用serviceName服务的所有peers
	peers *PeerList
	// 
	handler Handler
	logger Logger
	// 数据统计
	statsReporter StatsReporter
}

// 一个channel可以调用多个服务，并维护管理subchannel的所有peerList
type subChannelMap struct {
	sync.RWMutex
	subchannels map[string]*SubChannel
}

// 创建SubChannel，它继承了channel的所有peerList。
// 即channel中的subchannelMap每个subchannel中的peerList全部一样。
func newSubChannel(serviceName string, ch *Channel) *SubChannel {
	return &SubChannel{
		serviceName: serviceName,
		peers: ch.peers,
		topChannel: ch,
		handler: &handlerMap{}, // 默认处理方法
		logger: logger,
		statsReporter: ch.StatsReporter(), // 数据统计报告interface
	}
}

// 返回subChannel调用对方的服务名
func (c *SubChannel) ServiceName() string {
	return c.serviceName
}

// 准备发起一个调用，需要填充tchannel协议头部
// 根据subchannel指定的serviceName，同时获取一个peer
// 最后通过Peer填充协议头部
func (c *SubChannel) BeginCall(
	ctx context.Context, 
	methodName string,
	callOptions *CallOptions) (
		*OutboundCall, error
	){
	if callOptions == nil {
		callOptions = defaultCallOptions
	}
	
	// 获取没有被之前选中的peer, 由它发起一个rpc调用
	peer, err := c.peers.Get(callOptions.RequestState.PrevSelectedPeers())
	if err !=nil{
		return nil, err
	}
	
	// 前面章节介绍过，它会填充tchannel协议头。从该peer中获取一个可用的connection，调用serviceName服务
	return peer.BeginCall(ctx, c.ServiceName(), methodName, callOptions)
}

// 返回该channel与远程serviceName建立连接的所有peerList
//
func (c *SubChannel) Peers() *PeerList {
	return c.peers
}

// 如果是通过Isolated方法建立的SubChannel, 则peers是独立的，与channel不同
func (c *SubChannel) Isolated() bool {
	c.RLock()
	defer c.RUnlock()
	return c.topChannel.Peers() != c.peers
}

// 注册local service提供的服务
// local service通过handlerMap对象存储本地服务的methodName与handler
// 当外部请求进来时，则通过请求参数methodName获取handler，并做相应的处理
func (c *SubChannel) Register(h Handler, methodName string) {
	handlers, ok := c.handler.(*handlerMap)
	if !ok {
		panic()
	}
	handlers.register(h, methodName)
}

// 拷贝local service注册的所有服务, 快照
func (c *SubChannel) GetHandlers() map[string]Handler {
	handlers, ok := c.handler.(*handlerMap)
	if !ok {
		panic(...)
	}
	
	handlers.RLock()
	handlersMap := make(map[string]Handle, len(handlers.handlers))
	for k, v := range handlers.handlers {
		handlersMap[k] = v
	}
	handlers.RUnlock()
	return handlersMap
}

// 设置local service对外提供的服务处理
func (c *SubChannel) SetHandler(h Handler) {
	c.handler = h
}

// 获取该channel的所有tags
func (c *SubChannel) StatsTags() map[string]string {
	tags := c.topChannel.StatsTags()
	tags["subchannel"] = c.serviceName
	return tags
}

// 在channel的subChannelMap中注册一个新的subChannel
func (subChMap *subChannelMap) registerNewSubChannel(
	serviceName string, 
	ch *Channel) (
		_ *SubChannel,
		added bool,
	){
	subChMap.Lock()
	defer subChMap.Unlock()
	
	if subChMap.subchannels == nil{
		subChMap.subchannels = make(map[string]*SubChannel)
	}
	
	if sc, ok := subChMap.subchannels[serviceName]; ok {
		return sc, false
	}
	
	sc := newSubChannel(serviceName, ch)
	subChMap[serviceName] = sc
	return sc, true
}

// 通过remote service的serviceName，获取subChannel
func (subChMap *subChannelMap) get(serviceName string) (*SubChannel, bool) {
	subChMap.RLock()
	sc, ok := subChMap.subchannels[serviceName]
	subChMap.RUnlock()
	return sc, ok
}

// 获取serviceName对应的subChannel，如果不存在，则注册remote service
func (subChMap *subChannelMap) getOrAdd(
	serviceName string, 
	ch *Channel) (
	_ *SubChannel, 
	added bool
){
	if sc, ok := subChMap.get(serviceName); !ok {
		return sc, false
	}
	
	return subChMap.registerNewSubChannel(serviceName, ch)
}

// 这里只针对Isolated的subChannel
// 这里我们可以看到它是对channel的subChannelMap所有subChannel中的peer进行更新。
func (subChMap *subChannelMap) updatePeer(p *Peer) {
	subChMap.RLock()
	for _, subCh := range subChMap.subChannels {
		if subCh.Isolated() {
			subCh.RLock()
			subCh.Peers().onPeerChange(p)
			subCh.RUnlock()
		}
	}
	subChMap.RUnlock()
}
```

# 总结

这里说明下channel、subchannel与peerList之间的关系，再不说大家可能会看得有些累

一个channel是指发起调用的serviceName，如果这个channel要调用其他微服务，则需要对调用的微服务建立一个subChannelMap存储结构，每个微服务对应一个subChannel, 这里的每个subChannel中都有一个远程服务serviceName，那么这里我们就可以把rpc调用的双方建立起连接。

然后对于远程serviceName，可能会存在多个相同服务实例，对应不同的IP:PORT。这样就需要对subChannel细化，这就产生了subChannel的peerList存储结构，它里面有peersByHostPort，由此，channel 本地serviceName通过channel、subChannel和peerList， 就可以知道local service调用remote service的IP:PORT，以及针对所有连接的管理。


有个关于新建subChannel的疑问，我们可以从代码看到subChannel的Peers是从channel中拷贝的，而subchannel是针对指定remote service服务的，如果直接拷贝channel的peers，则PeersByHostPort并不只是subChannel指定服务的所有IP:PORT， 那映射的意义就有问题？除非全部是Isolated

如果上个疑问是没有问题的，则前面的结论则是有些不合理的

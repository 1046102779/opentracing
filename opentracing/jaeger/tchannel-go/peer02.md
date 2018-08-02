# PeerList

我们继续第二讲阅读源代码Peer

```shell
// GetNew方法，目的是找出最新创建的Peer.
// 具体做法：对比当前PeerList与上一个PeerList，做差集。得到的结果就是最新的Peer
// notice：一般来说做差集得到的应该是一个列表。但是这个操作是针对两个相邻的数据，相差1个peer
// 所以遇到prevSelected不存在的Peer，则就是最新的
func (l *PeerList) GetNew(prevSelected map[string]struct{}) (*Peer, error) {
	l.Lock()
	defer l.Unlock()
	
	if l.peerHeap.Len() == 0 {
		return nil, ErrNoPeers
	}
	
	peer := l.choosePeer(prevSelected, true)
	if peer == nil {
		peer = l.choosePeer(prevSelected, false)
	}
	
	if peer == nil {
		return nil, ErrNoNewPeers
	}
	return peer, nil
}
```

这个有个小问题，l.choosePeer方法可能会执行两次，第一次传入true，再次传入false。大家进入这个choosePeer就会知道，它的范围应该是扩大。但是这里bool值传反了。不然第一个方法peer为nil，则第二个方法返回值必为nil。

同时tchannel-go也存在map的竞态问题，我也提过相关的PRs，作者说目前这个项目不再计划更新，已暂停。所以相关的问题也不会在关注了。例如：[pr 710](https://github.com/uber/tchannel-go/pull/710)

```shell
//  首先通过GetNew方法获取peer, 失败则直接获取heap上的第一个peer
func (l *PeerList) Get(prevSelected map[string]struct{}) (*Peer, error) {
	peer, err := l.GetNew(prevSelected)
	if err == ErrNoNewPeers {
		l.Lock()
		// 直接获取PeerList上peerHeap的第一个数据即可
		peer = l.choosePeer(nil, false)
		l.Unlock()
	} else if err != nil {
		return nil, err
	}
	
	if peer == nil {
		return nil, ErrNoPeers
	}
	return peer, nil
}

// 删除channel中PeerList的peer
func (l *PeerList) Remove(hostPort string) error {
	l.Lock()
	defer l.Unlock()
	
	// 如果PeerList不存在hostPort的peer
	p, ok := l.peersByHostPort[hostPort]
	if !ok {
		return ErrPeerNotFound
	}
	
	// 存在，则删除与Peer相关的所有信息
	
	// PeerList所在channel引用减一
	p.delSC()
	delete(l.peersByHostPort, hostPort)
	// 最小堆删除PeerList peerHeap中的peer
	l.peerHeap.removePeer(p)
	
	return nil
}

// 做差集，然后返回一个作为最新的Peer。
// 通过container/heap标准库实现。还有container/list和container/ring两个package
// 其实就是选出最新的Peer后，并插入到heap末尾，这个heap是最小堆。它是以score值进行由小到大的有序
//
// 为了拿到一个在prevSelected不存在的peer，choosePeer方案：从最小堆peerHeap中遍历pop Peer，并在prevSelected校验是否存在，并最后再把取出来的所有Peer push到最小堆中。
// 我觉得这个性能太差了，为什么不用PeerList中的peersByHostPort作为校验?
func (l *PeerList) choosePeer(prevSelected map[string]struct{}, avoidHost bool) *Peer {
	canChoosePeer := func(hostPort string) bool{
		if _, ok := prevSelected[hostPort]; ok {
			return false
		}
		
		if avoidHost {
			if _, ok := prevSelected[getHost(hostPort)]; ok {
				return false
			}
		}
		return true
	}
	... // others
}

// GetOrAdd方法也很啰嗦
// 方法中的l.exists已经校验过一次是否存在peer了
// 然后l.Add又进行了两次校验，所以说tchannel-go还是有很多代码或者逻辑混乱的局面
// 
// 以后如果有时间，自己理解整个tchannel-go框架
// 想替换掉Hyperbahn和tchannel-go中的thrift协议，改为etcd和protobuffer协议
// 所以该方法中的l.exists代码可以去掉。
func (l *PeerList) GetOrAdd(hostPort string) *Peer {
	// 判断PeerList的PeersByHostPort是否已存在Peer
	if ps, ok := l.exists(hostPort); ok {
		return ps.Peer
	}
	// 不存在，则在PeerList添加Peer相关信息
	return l.Add(hostPort)
}

// 拷贝一份PeerList中的PeersByHostPort数据
// listCopy的初始化，应该要放在锁操作的外面更好
func (l *PeerList) Copy() map[string]*Peer {
	l.RLock()
	defer l.RUnlock()
	
	listCopy := make(map[string]*Peer)
	for k, v := range l.peersByHostPort {
		listCopy[k] = v.Peer
	}
	return listCopy
}

// 返回Peers的长度， 两种方法都可以
// 1. 通过peerHeap的slice长度
// 2. peersByHostPort的长度
func (l *PeerList) Len() int {
	...
}

// getPeerScore用来获取peerScore，存在三个问题：
// 1. PeerList中的exists方法和该方法含义相同，无需再要这个方法
// 2. 读取PeerList中的PeersByHostPort数据，没有加上读锁。如果继续往下看代码，你会发现这个读锁放在了方法外面，上文已提到过这个锁设计缺陷。
// 3. PeerScore的score不应该直接在方法中返回，它属于PeerScore的子元素，所以应该是由它的行为提供；或者因为都在tchannel package中，所以也可以直接读取score数据
func (l *PeerList) getPeerScore(hostPort string) (*peerScore, uint64, bool) {
	ps, ok := l.peersByHostPort[hostPort]
	if !ok {
		return nil, 0, false
	}
	return ps, ps.score, ok
}

// 当PeerList上的Peer发生变化时，会有相应的事件触发，修改peer的score值
// 以及更新peerHeap中的peer
func (l *PeerList) onPeerChange(p *Peer) {
	... // 获取peer和它的score
	
	newScore := sc.GetScore(ps.Peer)
	if newScore == psScore {
		return
	}
	
	// PeerList的PeersByHostPort和peerHeap数据，以及因为score的变化，堆的重新调整
	// 这里我们重点注意下l.updatePeer方法和l.peerHeap.updatePeer
	l.Lock()
	l.updatePeer(ps, newScore)
	l.Unlock()
	return
}

// 由于score值的变化，更新PeersByHostPort和peerHeap
func (l *PeerList) updatePeer(ps *peerScore, newScore uint64) {
	if ps.score == newScore {
		return
	}
	
	// 由于peerScore是从PeersByHostPort取出的指针对象，所以直接赋值score，底层地址存放的变量值也是会发生变化的
	ps.score = newScore
	// 再看peerHeap根据score值调整ps的位置
	l.peerHeap.updatePeer(ps)
}

// 这里涉及到了peerHeap，连带讲了算了
// 咋一看，卧槽，调整个毛线，啥也没变
// 
// 你再看看PeersByHostPort和peerHeap的增加、修改相关操作。然后你就明白两个对象的底层PeerScore是共享一块内存的，所以PeerScore的变化，是能够互相影响的，前面已经对底层PeerScore的score进行了调整，所以可以直接调整heap了。
func (ph *peerHeap) updatePeer(peerScore *peerScore) {
	heap.Fix(ph, peerScore.index)
}
```

# peerScore

```shell
// channel管理的rpc peer
type Peer struct {
	sync.RWMutex
	
	// Connectable interface用于rpc连接建立，在new channel时传入。
	channel Connectable
	// Peer本身的hostPort
	hostPort string
	// 当Peer的属性发生变化时，调用。例如：当Peer的score值变动，则会调整最小堆peerHeap
	onStatusChanged func(*Peer)
	// 当Peer关闭连接时，需要移除关闭所有连接，并在PeerList也去掉Peer
	onClosedConnRemoved func(*Peer)
	onUpdate func(*Peer)
	
	// 对于onStatusChanged和onClosedConnRemoved两个事件方法的设计，有点设计问题
	// 因为在C++语言中，表示这个方法传入了两个参数，第一个参数自身*Peer，第二个参数传入的*Peer, 且这两个参数值是相等的
	
	// subchannel的数量
	scCount uint32
	
	newConnLock sync.Mutex
	
	// inboundConnections：作为被调用方的连接数
	inboundConnections []*Connection
	// outboundConnections：作为调用方的连接数
	outboundConnections []*Connection
	
	// 每次使用choosePeer方法时，被选中的该Peer次数
	chosenCount atomic.Uint64
}

// 新建Peer
func newPeer(
	channel Connectable, 
	hostPort string, 
	onStatusChanged  func(*Peer), 
	onClosedConnRemoved func(*Peer)) *Peer {
	... // check params
	
	// nil，则使用默认的空操作
	// 
	// 那如果onClosedConnRemoved是nil呢？
	if onStatusChanged == nil {
		onStatusChanged = noopOnStatusChanged
	}
	
	return &Peer{
		channel: 			channel,
		hostPort: 			hostPort,
		onStatusChanged:	onStatusChanged,
		onClosedConnRemoved: onClosedConnRemoved,
	}
}


// 携带score的Peer
type peerScore struct {
	*Peer
	
	// 根据PeerList中的ScoreCalculator计算分数值
	score uint64
	
	// index表示peerHeap最小堆中的索引位置
	index int
	
	// 当两个Peer的score相等时，它用来比较大小。具体后面看 ::TODO
	order uint64
}

func newPeerScore(p *Peer, score uint64) *peerScore{
	return &peerScore{
		Peer: p,
		score: score,
		index: -1,
	}
}

// 获取peer中的第i个连接，这里注意一点：
// 
// 这里的i值比较有意思，它把inboundConnections和outboundConnections连接起来
// 当i索引值小于inbound的长度时，则表示被调用的第i个连接；
// 否则，表示调用方的第i-len(inbound)个连接。
//
// 这里也需要加锁，只是加到了外层
func (p *Peer) getConn(i int) *Connection {
	inboundLen := len(p.inboundConnections)
	if i < inboundLen {
		return p.inboundConnections[i]
	}
	
	return p.outboundConnections[i-inboundLen]
}

// 上面的getConn就是为了getActiveConnLocked方法使用的，所以我觉得可以不要新增这个方法，而是直接使用闭包函数
//
// 该方法用于获取一个连接，不管这个连接所在的Peer是调用方还是被调用方。
// 这里的一个取巧，与RPC的服务发现有些策略类似——随机策略
// 当大量调用时，这种随机策略落的点分布可能比较均衡。
//
// 这个虽然方法名是getActiveConnLocked，但是实际上并没有在内部上锁，而是放在了方法外部。
// 所以看tchannel-go整个框架，锁的设计真的是乱
func (p *Peer) getActiveConnLocked() (*Connection, bool) {
	allConns := len(p.inboundConnections) + len(p.outboundConnections)
	if allConns == 0 {
		return nil, false
	}
	
	startOffset := peerRng.Intn(allConns)
	for i:=0; i < allConns; i++{
		connIndex := (i + startOffset) % allConns
		if conn := p.getConn(connIndex); conn.IsActive() {
			return conn, true
		}
	}
	
	return nil, false
}

// 获取一个active connection的源头，这里加锁
// 再也没有其他地方使用，所以这里按道理也是闭包函数
//
// 而这里没有使用Locked，上一个却使用了Locked
// 这就是该写Locked的地方，没有写；不该写Locked的地方，却写了Locked
func (p *Peer) getActiveConn() (*Connection, bool) {
	p.RLock()
	conn, ok := p.getActiveConnLocked()
	p.RUnlock()
	
	return conn, ok
}

// 如果没有active connection，则直接发起一个连接
// 如果连接是新建立的，则应该要保存在Peer的outboundConnections中。后续再看 ::TODO
//
// 这种获取Connection的方式在上一节的后记中有详细介绍
func (p *Peer) GetConnection(ctx context.Context) (*Connection, error) {
	if activeConn, ok := p.getActiveConn(); ok {
		return activeConn, nil
	}
	
	p.newConnLock.Lock()
	defer p.newConnLock.Unlock()
	
	if activeConn, ok := p.getActiveConn(); ok {
		return activeConn, nil
	}
	
	return p.Connect(ctx)
}

// getConnectionRelay方法和上一个方法差不多，区别在于它在上下文传播中携带了一些信息。
// 
// 看文档好像是为了避免连接被调用方和pyperbabn有流量进来。具体后面再看 ::TODO
func (p *Peer) getConnectionRelay(timeout time.Duration) (*Connection, error) {
	if conn, ok := p.getActiveConn(); ok {
		return conn, nil
	}
	
	p.newConnLock.Lock()
	defer p.newConnLock.Unlock()
	
	if activeConn, ok := p.getActiveConn(); ok {
		return activeConn, nil
	}
	
	ctx, cancel := NewContextBuilder(timeout).HideListeningOnOutbound().Build()
	defer cancel()
	
	return p.Connect(ctx)
}

// 对peer的subchannel数量计数+1操作，用于引用计数
func (*Peer) addSC() {
	p.Lock()
	p.scCount++
	p.Unlock()
}

// 对peer的subchannel数量计数-1操作，用于引用计数
func (*Peer) delSC() {
	p.Lock()
	p.scCount--
	p.Unlock()
}

// 如果Peer的调入和调出的连接都为0，且Peer的subchannel数量为0，则可以在PeerList中删除该Peer
func (p *Peer) canRemove() bool {
	p.RLock()
	count := len(p.inboundConnections) +
		len(p.outboundConnections) +
		int(p.scCount)
	p.RUnlock()
	
	return count
}

// 添加新的connection，通过inbound和outbound方向，添加到Peer中的inboundConnections或者outboundConnections中, 最后触发Peer属性变化的事件行为
//
// 因为connectionFor返回的是Peer的inboundConnection或者outboundConnections的指针地址
// 所以对conns的修改，就是对Peer底层连接的修改
func (p *Peer) addConnection(c *Connection, direction connectionDirection) error {
	conns := p.connectionFor(direction)
	
	if c.readStat() != connectionActive() {
		return ErrInvalidConnectionState
	}
	
	p.Lock()
	*conns = append(*conns, c)
	p.Unlock()
	
	p.onStatusChanged(p)
	
	return nil
}

// 从connsPtr中移除changed connection
// 这里的conns复制了一份，没必要
//
// 采用的策略，经常看到：
// 对于slice无序数据删除某个元素，操作步骤：
// 1. 找到该元素的索引index
// 2. 和尾部元素交换
// 3. 并删除尾部元素
func (p *Peer) removeConnection(connsPtr *[]*Connection, changed *Connection) bool {
	conns : = *connsPtr
	for i, c := range conns {
		if c == changed {
			last := len(conns) - 1
			conns[i], conns[last] = conns[last], nil
			*connsPtr = conns[:last]
			return true
		}
	}
	
	return false
}

// 移除connection
func (p *Peer) connectionCloseStateChange(changed *Connection) {
	if changed.IsActive() {
		return
	}
	
	// 试图并分别从peer中的inboundConnections和outboundConnections中移除changed
	p.Lock()
	found := p.removeConnection(&p.inboundConnections, changed)
	if !found {
		found = p.removeConnection(&p.outboundConnections, changed)
	}
	p.Unlock()
	
	// 如果存在changed, 则调用关闭连接和状态变化事件
	if found {
		p.onClosedConnRemoved(p)
		p.onStatusChanged(p)
	}
}

// 建立outbount连接，调用方，调出
func (p *Peer) Connect(ctx context.Context) (*Connection, error) {
	return p.channel.Connect(ctx, p.hostPort)
}

// 它从Peer的inboundConnections和outboundConnections获取一个active connection
// 并通过这个connection 发起一个rpc调用, 具体后面再看 ::TODO
func (p *Peer) BeginCall(
	ctx context.Context, 
	serviceName, methodName string, 
	callOptions *CallOptions) (*OutboundCall, error) {
		if callOptions == nil {
			callOptions = defaultCallOptions
		}
		
		callOptions.RequestState.AddSelectedPeer(p.HostPort())
		
		// 校验调用是否合法
		if err := validateCall(ctx, serviceName, methodName, callOptions); err !=nil {
			return nil, err
		}
		
		// 从peer中获取一个active connection
		conn, err := p.GetConnection(ctx)
		if err !=nil{
			return nil, err
		}
		
		// 填充tchannel协议的通用部分
		call , err :=  conn.beginCall(ctx, serviceName, methodName, callOptions)
		if err !=nil{
			return nil, err
		}
		
		return call, err
}

// 获取Channel与该Peer建立连接的总数量，并以调入和调出分开统计
func (p *Peer) NumConnections() (inbound int, outbound int){
	p.RLock()
	inbound = len(p.inboundConnections)
	outbound = len(p.outboundConnections)
	p.RUnlock()
	return
}

// 该方法返回channel与该Peer建立的所有连接调出消息交换总数量
// 
// 对于channel调出Peer的消息总数为inboundConnections所有连接的调出方，
// 加上outboundConnections所有连接的调出方。
func (p *Peer) NumPendingOutbound() int {
	count := 0
	
	p.RLock()
	for _, c:= range p.outboundConnections {
		count += c.outbound.count()
	}
	
	for _, c := range p.inboundConnections {
		count += c.outbound.count()
	}
	p.RUnlock()
	
	return count
}

// 对Channel与Peer所有连接执行f(connection)方法, 对connection的操作，影响Peer的所有connections
func (p *Peer) runWithConnections(f func(*Connection)) {
	p.RLock()
	for _, c := range p.inboundConnections {
		f(c)
	}
	
	for _, c := range p.outboundConnections {
		f(c)
	}
	p.RUnlock()
}

// 更新操作Peer
func (p *Peer) callOnUpdateComplete() {
	p.RLock()
	f := p.onUpdate
	p.RUnlock()
	
	if f != nil {
		f(p)
	}
}
```

# 总结

首先搞清楚一点，在Channel中的PeerList和Peer都是作为remote service peer。也就是说channel上的所有PeerList的peer，都是与该channel已建立的连接列表，作者区分了调入和调出两类连接：inbound作为channel的调用方，而对于PeerList的Peer则是被调用方，即rpc调用的service；outbound作为Peer的调用方，也即发起rpc调用的client；

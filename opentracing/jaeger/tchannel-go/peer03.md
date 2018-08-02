# RootPeerList

RootPeerList是作为PeerList的parent，而Channel存储的是PeerList。每个PeerList都有一个parent。

```shell
type RootPeerList struct {
	sync.RWMutex
	
	channel Connectable
	onPeerStatusChanged func(*Peer)
	peersByHostPort map[string]*Peer
}

// 新建RootPeerList对象
func newRootPeerList(ch Connectable, onPeerStatusChanged func(*Peer)) *RootPeerList {
	return &RootPeerList{
		channel: ch,
		onPeerStatusChanged: onPeerStatusChanged,
		peersByHostPort: make(map[string]*Peer),
	}
}

// 根据RootPeerList对象，创建子对象PeerList
func (l *RootPeerList) newChild() *PeerList {
	// PeerList的scoreCalculator默认分值计算方法
	return newPeerList(l)
}

// RootPeerList对象中的peersByHostPort属性
// 添加Peer
// 新建Peer，并把RootPeerList对象中的channel、onPeerStatusChanged和onClosedConnRemoved方法赋值给Peer对象
//
// 前半段代码可以通过l.Get(hostPort)方法替代
func (l *RootPeerList) Add(hostPort string) *Peer {
	l.RLock()
	
	if p, ok := l.peersByHostPort[hostPort]; ok {
		l.RUnlock()
		return p
	}
	l.RUnlock()
	
	l.Lock()
	defer l.Unlock()
	
	if p, ok := l.peersByHostPort[hostPort]; ok {
		return p
	}
	
	var p *Peer
	p = newPeer(l.channel, hostPort, l.onPeerStatusChanged, l.onClosedConnRemoved)
	l.peersByHostPort[hostPort] = p
	return p
}

// 如果RootPeerList对象不存在，则增加hostPort；否则直接返回Peer
// 
// 其中的l.Get代码片段是和l.Add方法存在冗余
func (l *RootPeerList) GetOrAdd(hostPort string) *Peer {
	peer, ok := l.Get(hostPort)
	if ok {
		return peer
	}
	
	return l.Add(hostPort)
}

// 通过hostPort，获取Peer对象
func (l *RootPeerList) Get(hostPort string) (*Peer, bool) {
	l.RLock()
	p, ok := l.peersByHostPort[hostPort]
	l.RUnlock()
	return p, ok
}

// 关闭连接时，移除Peer
func (l *RootPeerList) onClosedConnRemoved(peer *Peer) {
	// 从RootPeerList对象中获取hostHost对应的Peer
	hostPort := peer.HostPort()
	p, ok := l.Get(hostPort)
	if !ok {
		return
	}
	
	// 如果Peer中的调入、调出的connections和subchannel计数累积和为0，则表示可以删除Peer了
	if p.canRemove() {
		l.Lock()
		delete(l.peersByHostPort, hostPort)
		l.Unlock()
		l.channel.Logger().WithFields(
			...
		)
	}
	return
}

// 获取RootPeerList对象的所有peersByHostPort，快照一份
func (l *RootPeerList) Copy() map[string]*Peer {
	l.RLock()
	defer l.RUnlock()
	
	listCopy := make(map[string]*Peer)
	for k, v := range l.peersByHostPort {
		listCopy[k] = v
	}
	return listCopy
}
```

# PeerHeap

PeerHeap用于维护PeerScore列表的最小堆, 并保证PeerScore存储的有序

```shell
// peerHeap最小堆， peerScore最小堆各个节点
//
// 实现heap最小堆，就需要实现container/heap的Interface接口
type Interface {
	sort.Interface
	Push(x interface{})
	Pop() interface{}
}

type peerHeap struct {
	peerScores []*peerScore
	rng		    *rand.Rand
	order 		uint64
}

func newPeerHeap() *peerHeap {
	return &peerHeap{rng: trand.NewSeeded()}
}

// peerHeap实现了sort Interface： 三个方法Len、Less和Swap
func (ph peerHeap) Len() int {
	return len(ph.peerScores) 
}

// 比较大小
// 当peerScore两个节点的score值不同时，直接比较大小
// 否则，比较peerScore的order
// 
// 至于order值是如何来的，后面再看 ::TODO
func (ph peerHeap) Less(i, j int) bool {
	if ph.peerScores[i].score == ph.peerScores[j].score {
		return ph.peerScores[i].order < ph.peerScores[j].order
	}
	return ph.peerScores[i].score < ph.peerScores[j].score
}

// 交换peerScore，并修改该对象属性index值，它的peerScores索引值变了
func (ph peerHeap) Swap(i, j int) {
	ph.peerScores[i], ph.peerScores[j] = ph.peerScores[j], ph.peerScores[i]
	ph.peerScores[i].index = i
	ph.peerScores[j].index = j
}

// 追加peerScore节点到peerScores列表后, 并记住索引位置
func (ph *peerHeap) Push(x interface{}) {
	n := len(ph.peerScores)
	item := x.(*peerScore)
	item.index = n
	ph.peerScores = append(ph.peerScores, item)
}

// 获取peerScores列表最后一个节点，并把该节点的index值设置为-1，为了防止取到其他节点。
func (ph *peerHeap) Pop() interface{} {
	if len(ph.peerScores) <=0 {
		return nil
	}
	
	item := ph.peerScores[len(ph.peerScores) -1 ]
	item.index = -1 // for safety
	ph.peerScores = ph.peerScores[:len(ph.peerScores)-1]
	return item
}

// 以上则实现了heap最小堆所需要的基础方法

// 所有的heap最小堆操作，都是操作index，底层的peerScore都是通过共享内存修改的
// 根据score分数值，调整最小堆中的节点位置
func (ph *peerHeap) updatePeer(peerScore *peerScore) {
	heap.Fix(hp, peerScore.index)
}

// 从heap最小堆中删除节点
func (ph *peerHeap) removePeer(peerScore *peerScore) {
	heap.Remove(hp, peerScore.index)
}

// 从最小堆中获取堆顶节点，并返回
func (ph *peerHeap) popPeer() *peerScore {
	return	heap.Pop(ph).(*peerScore)
}

// 理论上，只需要最小堆添加节点就行，但是这里对hp.Order进行的相关操作，以及peerScore的order也进行了相应计算
// 这个peerScore的order一般来说，新进来peerScore的order值一般都比较占有优势
// 具体后续再看 ::TODO
func (ph *peerHeap) pushPeer(peerScore *peerScore) {
	ph.order++
	newOrder := ph.order
   // randRange will affect the deviation of peer's chosenCount
   randRange := ph.Len()/2 + 1
   peerScore.order = newOrder + uint64(ph.rng.Intn(randRange))
   
	heap.Push(ph, peerScore)
}

// 交换peerScore的order，可能会影响排序
func (ph *peerHeap) swapOrder(i, j int) {
	if i == j{
		return
	}
	hp.peerScores[i].order, hp.peerScores[j].order = 
	hp.peerScores[j].order, ph.peerScores[i].order
	
	heap.Fix(hp, i)
	heap.Fix(hp, j)
}

// 添加一个peerScore节点到heap中，但是不理解为何要进行随机交换order排序?
// 
// 这个需要好好考虑下..... ::TODO
func (ph *peerHeap) addPeer(peerScore *peerScore) {
	ph.pushPeer(peerScore)
	
	r := ph.rng.Intn(ph.Len())
	ph.swapOrder(peerScore.Index, r)
}
```

# 总结

对于RootPeerList，我们可以看到PeerList的生成都是通过root生成child。并把root的相关方法, 例如：分数值计算方法等，传给child。其他都是对RootPeerList的peersByHostPort进行相关处理

对于heapPeer，主要是实现了container/heap所需要的最小子集。但是对于其中的order、以及addPeer一些概念和实现，不能很好的理解，我们后续再看 ::TODO

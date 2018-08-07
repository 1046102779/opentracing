# peer

peer中文翻译为：对等体, 它在tchannel-go中表示rpc调用或者被调用的一方。

peer相关错误码：

```shell
ErrInvalidConnectionState // 连接处于无效状态

ErrNoPeers  //  没有可用的peer

ErrPeerNoFound // 没有发现peer

ErrNoNewPeers // 没有新的可用peer

```

相关peer代码：

```shell
// peer使用该interface，创建connection。peer所拥有的channel实现了该interface
type Connectable interface {
	Connect(ctx context.Context, hostPort string) (*Connection, error)
	
	Logger() Logger
}

// channel拥有所有建立连接的Peers
type PeerList struct {
	sync.RWMutex
	
	// parent：后续补充
	parent *RootPeerList
	// channel所有连接，对于每个远端hostPort所对应的peer
	peersByHostPort map[string]*peerScore
	// peerHeap: 后续补充, 它基于peers的score，来维护peers得最小heap
	peerHeap *peerHeap
	// channel所有连接的peers score计算
	scoreCalculator ScoreCalculator
	// lastSelected: 后续补充
	lastSelected uint64
}

// 创建PeerList，并初始化。RootPeerList后续补充
func newPeerList(root *RootPeerList) *PeerList {
	reutrn &PeerList{
		parent: root,
		peersByHostPort: make(map[string]*peerScore),
		scoreCalculator: newPreferIncomingCalculator(),
		peerHeap: newPeerHeap(),
	}
}

// 设置channel所有连接peers的score计算方式
// 最后并重新计算当前channel下所有peers的score
func (l *PeerList) SetStrategy(sc ScoreCalculator) {
	l.Lock()
	defer l.Unlock()
	
	l.scoreCalculator = sc
	for _, ps := range l.peersByHostPort {
		newScore := l.scoreCalculator.GetScore(ps.Peer)
		// 更新peer的score
		l.updatePeer(ps, newScore)
	}
}

// 根据peerList创建一个它的兄弟PeerList。
func (l *PeerList) newSibling() *PeerList {
	sib := newPeerList(l.parent)
	return sib
}

// 在channel的PeerList中增加一个peer, 这个方法引出了后记
func (l *PeerList) Add(hostPort string) *Peer {
	// 如果hostPort已存在，则不需要增加peer
	if ps, ok := l.exists(hostPort); ok {
		return ps.Peer
	}
	
	l.Lock()
	defer l.Unlock()
	
	// 再次校验是否已经存在peer
	if p, ok := l.peersByHostPort[hostPort]; ok {
		return p.Peer
	}
	
	// 确定不存在peer，创建peer， 增加RootPeerList的peersByHostPort映射关系
	p := l.parent.Add(hostPort)
	// 增加subchannel的引用计数
	p.addSC()
	// 根据peer创建，peerScore对象
	ps := newPeerScore(p, l.scoreCalculator.GetScore(p))
	
	// 通过channel的peersByHostPort存储映射关系
	l.peersByHostPort[hostPort] = ps
	// push peer到peer heap中，后面再介绍
	l.peerHeap.addPeer(ps)
	
	return p
}
```

## 后记


### 第一个讨论点

我们在看tchannel-go源代码过程中经常遇到一个这样的写法，大家可以注意下：

`tips: 下面我们都假设YYY已经初始化过了。`

```shell
type XXX struct {
	sync.Mutex
	
	YYY map[string]interface{}
}

func (x *XXX) Add(key string, value interface{}) {
	x.RLock()
	if _, ok := x.YYY[key]; ok {
		return 
	}
	x.RUnlock()
	
	x.Lock()
	if _, ok := x.YYY[key]; ok {
		return
	}
	
	x.YYY[key]=value
	x.Unlock()
}
```

在其他项目中也会存在这种写法。首先对于存在竞态的数据，在读写操作时需要加锁互斥操作。所以，下面这种做法也是经常见到的。

```shell
func (x *XXX) Add(key string, value interface{}) {
	x.Lock()
	defer x.Unlock()

	if _, ok := x.YYY[key]; ok {
		return
	}
	
	x.YYY[key] = value
	return
}
```

对比这两者的写法，没有太多不同。这里要考虑的第一点：

**读锁和读写锁，时效性明显不同。对于读锁，只有在写时阻塞，可以并发读；对于读写锁，读写都互斥。后者明显时耗大一些，如果并发量够大，则效果更明显**

我们考虑第二点，为何第一种写法，在尝试读取后，下一个锁操作内还要取尝试取一次数据，失败然后再进行存储操作？

**因为每一次锁操作是一个完整操作，如果在临界区间，goroutine被调度，则其他想要操作该锁也得等待该锁释放，最后该goroutine被调度，完成释放锁后；如果该goroutine又被调度，则其他goroutine可能会操作该互斥YYY变量，所以下次进入临界区时，需要尝试再取一次数据，如果没有则进行存储操作**


### 第二个讨论点

对于PeerList的Add方法，或者说整个tchannel-go有一点做得不太好的地方是，方法里的锁操作太多，而且没有标识，比如`l.parent.Add`和`p.addSC`方法都存在锁操作，如果外界直接调用，锁相同则会导致死锁。在go有关的很多项目都是这样写的:

```shell
func (x *XXX) AddLock() {
	x.Lock()
	defer x.Unlock()
	... // other operations
}

func (x *XXX) AddNoLock() {
	... // other operations
}

或者

func (x *XXX) Add() {
	... // other operations
}
```

这个方法显示的告诉大家，方法内是否存在锁操作，这样尽量减少死锁反应。

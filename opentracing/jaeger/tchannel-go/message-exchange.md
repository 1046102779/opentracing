# message exchange

在前面的章节中，经常提及到message exchange概念，它用来表示一个peer与connection的消息交换。每个message exchange都有一个channel，它常常用于从这个peer上接受frames，当这个message exchange超时或者取消时，一个context上下文能够控制并作出其他相应的处理操作

```shell
// 从messageExchange结构体，我们可以看出通过recvCh管道接收Frame。
type messageExchange struct {
	recvCh chan *Frame
	errCh errNotifier
	// 如果context上下文超时或者取消，则做出相应动作
	ctx context.Context
	// 消息ID，每一次调用是一个message id
	msgID uint32
	// 帧消息类型：call req； call req continue；call res； call res continue,
	// call ping req; call ping res; call init req; call init res等
	msgType messageType
	// 下面有介绍
	mexset *messageExchangeSet
	framePool FramePool
	
	shutdownAtomic atomic.Uint32
	errChNotified atomic.Uint32
}

// 发送frame到connection上
func (mex *messageExchange) forwardPeerFrame(frame *Frame) error {
	... // check error
	select {
	// frame从Peer通过channel传输到connection上, 它会被goroutine recvMessage接收到
	case mex.recvCh <- frame:
		return nil
	// 增加recvCh buffer大小
	case <-mex.ctx.Done():
		return GetContextError(mex.ctx.Err())
	case <-mex.errCh.c:
		select {
		case mex.recvCh <- frame:
			return nil
		default:
		}
		return mex.errCh.err
	}
}

func (mex *messageExchange) checkFrame(frame *Frame) error {
	if rame.Header.ID != mex.msgID {
		... // err log
		return errUnexpectedFrameType
	}
	return nil
}

func (mex *messageFrame) recvPeerFrame() (*Frame, error){
	... // err handler
	
	select {
	// frame从connection传输到Peer上
	case frame := <-mex.recvCh:
		if err := mex.checkFrame(frame); err != nil{
			return nil, err
		}
		return frame, nil
	case <-mex.ctx.Done():
		return nil, GetContextError(mex.ctx.Err())
	case <- mex.errCh.c:
		...
	}
}

// message exchange通知关闭通道，并从message exchange set中移除message id
func (mex *messageExchange) shutdown() {
	if !mex.shutdownAtomic.CAS(0, 1) {
		return
	}
	
	if mex.errChNotified.CAS(0, 1) {
		mex.errCh.Notify(errMexShutdown)
	}
	
	mex.mexset.removeExchange(mex.msgID)
}

// 流入的message exchange数据通道超时
func (mex *messageExchange) inboundExpired() {
	mex.mex.expireExchange(mex.msgID)
}
```

# message exchange set

它主要用于把frame从peer路由到一个合适的messageExchange上；每一个connection都有两个messageExchangeSets，一个用于outbound，另一个用于inbound；具体类型的handlers负责注册message exchange和从对应的messageExchangeSet移除。

```shell
type messageExchangeSet struct {
	sync.RWMutex
	
	log Logger
	name string
	onRemoved func()
	onAdded func()
	// 一个服务的消息通道列表全部清除之前，goroutine不能退出
	sedChRefs sync.WaitGroup
	
	// 每个message id对应一个messageExchange；所以同一个messageExchange是一次rpc调用
	exchanges map[uint32]*messageExchange
	// 这个map[msgID]struct{}使用场景太多了
	// 校验msgID所对应的message exchange是否超时
	expiredExchanges map[uint32]struct{}
	// 校验message exchange通道是否关闭
	shutdown bool
}

// 很有意思的地方：messageExchange存在元素messageExchangeSet；同时messageExchangeSet存在元素messageExchange；
func newMessageExchangeSet(log logger, name string) *messageExchangeSet {
	return &messageExchangeSet {
		name: name,
		log: ... // exchange name log
		exchanges: make(map[uint32]*messageExchange),
		expiredExchanges: make(map[uint32]struct{}),
	}
}

// 增加一个peer到connection的消息通道，用于一次的rpc调用数据传输
func (mexset *messageExchangeSet) addExchange(mex *messageExchange) error {
	// 如果一次rpc调用完成，则这个通道会关闭。并返回message exchange的错误
	if mexset.shutdown {
		return errMexSetShutdown
	}
	
	// 校验message id的消息通道是否已存在, 存在则返回错误
	if _, ok := mexset.exchanges[mex.msgID]; ok {
		return errDuplicateMex
	}
	
	// 不存在则增加通道
	mexset.exchanges[mex.msgID] = mex
	mexset.sendChRefs.Add(1)
}

// 创建一个messageExchange，并添加到message exchange set中
func (mexsest *messageExchangeSet) newExchange(
	ctx context.Context, 
	framePool FramePool, 
	msgType messageType, 
	msgID uint32, 
	bufferSize int) (*messageExchange, error){
	
	// 创建一个message exchange
	mex := &MessageExchange{
		msgType: msgType,
		msgID: msgID,
		ctx: ctx,
		recvCh: make(chan *Frame, bufferSize),
		errCh: newErrNotifier(),
		mexset: mexset,
		framePool: framePool,
	}
	
	// 加锁添加messageExchange
	mexset.Lock()
	addErr := mexset.addExchange(mex)
	mexset.Unlock()
	
	mexset.onAdded()
	
	return mex, nil
}

// delete message exchange ; 外围需要加锁操作
func (mexset *messageExchangeSet) deleteExchange(msgID uint32) (found, timeout bool){
	// 如果msgID所对应的message exchange存在，则删除并返回
	if _, found := mexset.exchanges[msgID]; found {
		delete(mexset.exchanges, msgID)
		return true, false
	}
	
	// 如果不存在，则查看exchange message是否已超时，超时则返回不存在，且超时操作
	// 它的作用：后面再看 ::TODO
	if _, expired := mexset.expiredExchanges[msgID]; expired {
		delete(mexset.expiredExchanges, msgID)
		return false, true
	}
	
	return false, false
}

// 删除msgID
func (mexset *messageExchangeSet) removeExchange(msgID uint32) {
	// 删除msgID
	mexset.Lock()
	founc, expired := mexset.deleteExchange(msgID)
	mexset.Unlock()
	
	// message exchange移除时，waitGroup的变量减一操作；当为0时，goroutine直接退出；
	mexset.sendChRefs.Done()
	mexset.onRemoved()
}

// 删除message exchange， 并设置message exchange超时
func (mexset *messageExchangeSet) expireExchange(msgID uint32) {
	mexset.Lock()
	found, expired := mexset.deleteExchange(msgID)
	if found || expired {
		mexset.expireExchanges[msgID] = struct{}{}
	}
	mexset.Unlock()
	
	mexset.onRemoved()
}

// 等待message exchange清除完成，goroutine退出
func (mexset *messageExchangeSet) waitForSendCh() {
	mexset.sedChRefs.Wait()
}

// 统计该连接的当前message exchanges总数量
func (mexset *messageExchangeSet) count() int {
	mexset.RLock()
	count := len(mexset.exchanges)
	mexset.RUnlock()
	
	return count
}

// forwardPeerFrame方法是把一个frame从peer发送到合适的message exchange
func (mexset *messageExchangeSet) forwardPeerFrame(frame *Frame) error {
	// 从message exchange set中获取message exchange
	mexset.RLock()
	mex := mexset.exchanges[frame.Header.ID]
	mexset.RUnlock()
	
	... // if mex == nil, return 
	
	mex.forwardPeerFrame(frame)
	return
}

// 获取exchange exchange set的快照
func (mexset *messageExchangeSet) copyExchanges() (
	shutdown bool, 
	exchanges map[uint32]*messageExchange) {
	// 如果message exchange set已经关闭
	if mexset.shutdown {
		return true, nil
	}
	
	exchangesCopy := make(map[uint32]*messageExchange, len(mexset.exchanges))
	for k, mex := range mexset.exchanges {
		exchangesCopy[k] = mex
	}
	
	return false, exchangesCopy
}

// 停止所有的message exchange进行数据传递, 
// 
// 它是通过message exchange set遍历 message exchange，然后通过信号量channel终止数据传递
func (mexset *messageExchangeSet) stopExchanges(err error) {
	mexset.Lock()
	shutdown, exchanges := mexset.copyExchanges()
	mexset.shutdown = true
	mexset.Unlock()
	
	if shutdown {
		return
	}
	
	// 遍历获取的快照message exchange，然后通过channel发送信号量停止数据传递
	for _, mex := range exchanges {
		if mex.errChNotified.CAS(0, 1) {
			mex.errCh.Notify(err)
		}
	}
}
```

# 总结

每个message exchange set都是针对一个服务而言，该服务上有多个连接。每个message exchange负责frame从peer传输到connection上。且每个message exchange都是一次RPC调用，用message id表示; 并通过channel控制这个这个服务的所有连接进行释放操作

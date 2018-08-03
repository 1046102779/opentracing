# peer分数计算策略

ScoreCalculator定义了计算score方法的interface, 默认的分数值实现包括：zeroCalculator的零分数值实现、leastPendingCalculator的调用次数总数量和preferPendingCalculator的调出次数总数量

```shell
// 分值计算方法
type ScoreCalculator interface {
	GetScore(p *Peer) uint64
}

// ScoreCalculatorFunc是一个分数值的计算适配器

// 类似于http中的ServeHTTP, 如下所示：
//****************************************************
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
与
func HandleFunc(pattern string, handler func(ResponseWriter, *Request))
//****************************************************

type ScoreCalculatorFunc func(p *Peer) uint64 

func (f ScoreCalculatorFunc) GetScore(p *Peer) uint64 {
	return f(p)
}


// 所有分数值都为0的ScoreCalculator interface实现
type zeroCalculator struct{}

func (zeroCalculator) GetScore(p *Peer) uint64 {
	return 0
}


// 调用次数越多，score越大的计算方法。获取channel与该Peer的所有调入和调出连接的message exchange总数量
// 
// 这里的math.MaxUint64有点不理解？ 后续再看::TOOD
type leastPendingCalculator struct{}

func (leastPendingCalculator) GetScore(p *Peer) uint64 {
	inbound, outbound := p.NumConnections()
	if inbound + outbound == 0 {
		return math.MaxUint64
	}
	
	return uint64(p.NumPendingOutbound())
}

// 所有调出的总数量score计算方法。
type preferIncomingCalculator struct{}

func  (preferIncomingCalculator) GetScore(p *Peer) uint64 {
	inbound, outbound := p.NumConnections()
	if inbound + outbound == 0 {
		return math.MaxUint64
	}
	
	numPendingOutbound := uint64(pNumPendingOutbound())
	if inbound == 0 {
		return math.MaxInt32 + numPendingOutbound
	}
	
	return numPendingOutbound
}
```

# idle sweep

idle sweep是一个goroutine定时任务，去检测connection空闲时间，并清除空闲时间过长的连接

```shell
type idleSweep struct {
	ch *Channel
	// 每个连接的最大空闲时间
	maxIdleTime time.Duration
	// 多长时间间隔进行检测
	idleCheckInterval time.Duration
	// 用于停止检测，goroutine退出
	stopCh chan struct{}
	// start表示该goroutine是否已经启动
	start bool
}

// 开启idle sweep检测goroutine
func startIdleSweep(ch *Channel, opts *ChannelOptions) *idleSweep {
	is := &idleSweep {
		ch: ch,
		maxIdleTime: opts.MaxIdleTime,
		idleCheckInterval: opts.IdleCheckInterval,
	}
	
	is.start()
	return is
}

// 接着上面的startIdleSweep方法，继续执行goroutine启动任务
// 主要是初始化idleSweep相关参数, 启动一个goroutine
func (is *idleSweep) start() {
	// 如果goroutine已经启动，或者定期检测时长间隔为空， 则直接返回
	if is.started || is.idleCheckInterval <=0 {
		return
	}
	
	... // log records
	
	is.started = true
	is.stopCh = make(chan struct{})
	
	go is.pollerLoop()
}

// 启动定时机制，并定时检测检测channel上的所有空闲连接
func (is *idleSweep) pollerLoop() {
	// 这里的timeTicker是使用的channel中channelConnectionCommon参数
	ticker := is.ch.timeTicker(is.idleCheckInterval)
	
	for {
		select {
		case <-ticker.C:
			// 到时间则检测超时空闲连接，并清除
			is.checkIdleConnections()
		case <- is.stopCh:
			// 如果外界有干预，则直接退出
			ticker.Stop()
			return
		}
	}
}

// 检测超时空闲连接，并清除
func (is *idleSweep) checkIdleConnections() {
	now := is.ch.timeNow()
	
	idleConnections := make([]*Connection, 0, 10)
	is.ch.mutable.RLock()
	// 获取channel的所有连接, 并拿着这个当前时间和连接上次活跃时间做差值，结果与maxIdleTime比较
	for _, conn := range is.ch.mutable.conns {
		idleTime := now.Sub(conn.getLastActivityTime())
		if idleTime >= is.maxIdleTime {
			idleConnections = append(idleConnections, conn)
		}	
	}
	is.ch.mutable.RUnlock()
	
	// 获取的超时空闲连接，全部清除掉
	for _, conn := range idleConnections {
		// 如果连接不是活跃的，则直接过滤掉； 否则，则直接关闭该连接
		if !conn.IsActive() {
			continue
		}
		
		conn.close(LogField{"reason", "Idle connection closed"})
	}
}
```

# 总结

对于idle sweep的这种goroutine启动机制，并做定时任务这种写法，是一种很标准的写法。总结步骤为：

1. 传入外部参数，并初始化一个goroutine所需要的参数；
2. 设置内部参数，并start，在start方法中，启动一个goroutine；
3. 在goroutine中，通过time.TimeTicker触发定时任务执行，并使用for循环和select阻塞机制；且设置有goroutine退出外部介入机制；
4. 外部传入goroutine退出信号，或者定期执行任务。



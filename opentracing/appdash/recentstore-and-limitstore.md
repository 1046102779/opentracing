Appdash除了提供PersistentStore可持久化文件存储外，还提供了基于内存的限制存储两种，RecentStore和LimitStore， 这两种的Span Recorder都有生命周期，到了一定时间或者大小，Appdash认为这些Recorder不会再使用，删除它们。

当然基于内存的Span Recorder存储肯定有很多弊端，比如：报表统计做不到全局，只能看近期；针对第一种RecentStore存储，它是以时间为维度，进行Span Recorder的过期删除工作。缺点：1. 延展性差；2. 报表具有局部性等；延展性差是指，当业务量小时，Span Recorder本身就不多，则生命周期内的Recorder就少，不聚集。另一个，当业务量猛增时，生命周期长，则周期内的Span Recorder就非常多，容易造成内存瓶颈；对于第二种LimitStore存储，它是以内存占用大小为维度，进行Span Recorder删除。它的缺点是当业务量猛增时，Span Recorder很容易突破设置的内存限制，则数据量不全，或者搜索traceid时，无法搜索到。


因为内存存储的Span Recorder都是有限制的，所以肯定会有Span Recorder删除行为，则Appdash提供了相关删除接口

```shell
type DeleteStore interface{
	// 因为是基于Store interface操作
	Store
	
	// 提供了Delete删除Span Recorder操作
	Delete(...ID) error
}
```
# RecentStore

```shell
type RecentStore struct {
	// 基于时间周期的SpanRecorder删除
	MinEvictAge time.Duration
	
	DeleteStore
	
	// 带有Span Recorder的生命周期管理
	created map[ID]int64
	
	// 上一次删除的时间
	lastEvicted time.Time
	
	mu sync.Mutex
}
// 由上可以看出，RecentStore是利用MinEvictAge和lastEvicted两个时间，来对Span Recorder进行生命周期的管理。

func (rs *RecentStore) Collect(id SpanID, anns ...Annotation) error) {
	... // 并发
	// 存储新来的Span Recorder
	if _, present := rs.created[id.Trace]; !present {
		rs.created[id.Trace] = time.Now().UnixNano()
	}
	// 删除过期的Span Recorder列表
	if time.Since(rs.lastEvicted) > rs.MinEvictAge {
		rs.evictBefore(time.Now().Add(-1*rs.MinEvictAge))
	}
	
	return rs.DeleteStore.Collect(id, anns...)
}

// 删除小于t时间的Span Recorder。
func (rs *RecentStore) evictBefore(t time.Time) {
	evictStart := time.Now()
	
	rs.lastEvicted = evictStart
	
	tnano := t.UnixNano()
	
	var toEvict []ID
	for id, ct := range rs.created{
		if ct < tnano {
			toEvict = append(toEvict, id)
			delete(rs.created, id)
		}
	}
	
	if len(toEvict) ==0 {
		return
	}
	
	// 删除内存中的Span Recorder列表
	go func(){
		rs.DeleteStore.Delete(toEvict...)
	}()
	return
}
```

由RecentStore的Collect可以看出，虽然删除和存储Span Recorder是异步操作。理论上应该是单独的goroutine去对所有收集到的Span Recorder的生命周期进行管理，否则，缺点两个：

1. 如果没有新的Span Recorder到来，则无法触发过期的Span Recorder删除。
2. 如果过期时间MinEvictAge设置得很小，则不断新到来的Span Recorder会不断触发goroutine操作，这个也是不合理的。

# LimitStore

```shell
type LimitStore struct {
	// 存储的Span Recorder最大数量
	Max int
	
	DeleteStore
	
	mu sync.Mutex
	
	// 带Max的Span Recorder生命周期管理
	traces map[ID]struct{}
	
	// 通过环状存储管理
	ring []int64
	
	// 下一个插入Span Recorder位置
	nextInsertIdx int
}
// 对于LimitStore存储，它是基于存储Span Recorder最大数量的维度来维护所有的Span Recorder。它是通过traces、ring和nextInsertIdx两个存储来维护的。traces用来表示trace是否存在；ring和nextInsertIdx用来表示环插入trace维护

其实，环traces的维护只需要ring数组和nextInsertIdx两个变量来维护，但是在ring数组中查找traceid太慢，所以引入了traces map结构存储，时间复杂度为o(1)

func (ls *LimitStore) Collect(id SpanID, anns ...Annotation) error {
	... // 并发
	if ls.ring ==nil {
		ls.ring = make([]int64, ls.Max)
		ls.traces = make(map[ID]struct{}, ls.Max)
	}
	
	// 获取traceid是否存在, 存在即更新
	if _, ok := ls.traces[id.Trace]; ok {
		return ls.DeleteStore.Collect(id, anns...)
	}
	
	// 如果ring环的下个插入位置不为0，则表示ring环满了，需要覆盖(删除并插入)
	if nextInsert := ls.ring[ls.nextInsertIdx]; nextInsert !=0 {
		old := ID(ls.ring[ls.nextInsertIdx])
		delete(r.traces, old)
		ls.DeleteStore.Delete(old)
	}
	
	// 插入
	ls.traces[id.Trace] = struct{}{}
	ls.ring[ls.nextInsertIdx] = int64(id.Trace)
	ls.nextInsertIdx = (ls.nextInsertIdx+1)%ls.Max
	return ls.DeleteStore.Collect(id, anns...)
}
```


由此可看，RecentStore和LimitStore都是基于DeleteStore实现，且DeleteStore interface都是在Store上添加了Delete方法实现。而MemoryStore持久化存储则不仅实现了本地文件存储，还实现了DeleteStore interface。所以RecentStore和LimitStore都是基于MemoryStore实现的。

MemoryStore的定期本地文件存储，策略不够丰富，目前只支持一种策略：内存全局写入文件，因为MemoryStore限制有写入磁盘的有效时间，一旦过期，则会导致有些trace无法落地磁盘；而且每次全局写入，持久化需要的时间过长。

其中RecentStore的设计和实现是存在缺陷的。

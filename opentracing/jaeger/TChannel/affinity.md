[Hyperbahn](https://github.com/uber-archive/hyperbahn) 是 Uber 开源的一套服务发现和路由系统，专门用于包含大量微服务的大规模系统，可以使服务间的发现和沟通非常简单和可靠。 目前该库已经不活跃了，不再更新。与etcd、zookeeper和consul等产品具有相同的作用。

# Affinity(亲和性策略)
```shell
|----------------------------------------------------------------------|
|    ____________            ____________          -------> worker..1 |
|   |           |           |           |         |                    |
|   | Hyperbahn |---------->| relay ring|-------->|-------> worker... |
|   |  server   |           |           |         |                    |
|   |___________|           |___________|         |-------> worker..N |
|                                                                      |
|----------------------------------------------------------------------|
						
						Hyperbahn服务与后端服务之间的关系
```

上面画的这幅图，我是在微信开放平台时了解到relay的，微信说是中继服务器，用来刷token和托管公众号的token的，这样web用户访问时，就不需要进行token获取了。托管业务也就直接可以操作相关微信业务了。

这个Hyperbahn relay应该也是同理，负责处理Hyperbahn与后端服务之间的连接管理。

**这样再翻译Affinity这篇文章，就是按照这个意思来了。不然比较难理解。**

一个服务对应一组服务实例。

对于每个服务，都会有很多实例以防止单个服务负载过高和容错，以及与Hyperbahn relays建立好的连接集——the affine relays（它是relay ring的子集，负责发送和响应指定service的请求数据）。这个最小可行的Hyperbahn全连接到service相关的the affinie relays上。然而随着时间的推移，这些relays可能会超过最大文件描述符的数量ulimit。下面的策略就是为了解决这个问题。

*minPeersPerWorker*值是每个服务worker连接数量的最小保证。也就是说一个服务worker至少有三个连接可以供Hyperbahn relay使用。还有一个*minPeersPerRelay*值，它是Hyperbahn relay至少连接到一个服务的worker数量。

**那么由此可见一个relay ring至少与一个服务建立的连接数量为：`minPeersPerWorker*minPeersPerRelay`**


*relayCount*值是指Hyperbahn relay已经与服务workers建立好的连接数量。是这个服务的近似`k`值

对于每个服务通过gossiping advertisement传送给relay ring中的每个worker成员。每个Hyperbahn relay知道所关联服务worker的host:worker， *workerCount*是这个服务的worker数量，并且被每个相关的relay所知道。

**这里有几个概念，我感觉有点混：relay ring, Hyperbahn relay，affine relay**

这个relay ring可以通过排序所有的workers，并对每个worker给一个合适的位置`workerIndex`。

在relay ring中的每个位置有一个线性映射位置到每个worker ring。映射`relayIndex`到`workerIndex`的计算公式：

```shell
ratio = workerCount / relayCount

workerIndex = relayIndex *ratio
```

每个服务的worker数量应该为：

```shell
max(minPeersPerRelay, minPeersPerWorker * ratio)

minPeersPerRelay表示relay至少与服务workers建立的连接数，则至少有minPeersPerRelay个worker，否则这个条件无法满足。

minPeersPerWorker * ratio = 
minPeersPerWorker * (workerCount/relayCount)  =
minPeersPerWorker * workerCount / relayCount =  
```

每个relay都应该在worker组内映射对应的位置，然后连接到服务worker上，以确保每个relay和服务worker之间的最小连接数。此范围从worker ring的相应位置开始，但不包括range的末尾：

```shell
workerIndex = round(
	relayIndex * ratio +
	max(
		minPeersPerRelay,
		minPeersPerWorker * ratio,
	)
)
```

这种方法的一个期望特性是，这个affine relay集或者worker集的小变化应该在很大程度上，与先前的worker集重叠，保留了许多现有的连接。但是小的变化会影响每个relay的边界，并可能导致整个系统的调整。

请参与affinity.py以获取验证一系列方案的对等选择算法的静态属性模拟

## 疑问

应为这个Affinity没有给出相应的框架图，同时对relay ring, Hyperbahn relay，affine relay几个概念不清楚，所以感觉这个图和翻译有误。后面再改正

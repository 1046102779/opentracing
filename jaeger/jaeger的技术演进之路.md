jaeger uber的分布式跟踪系统，它是CNCF基金会的第15个项目。在16年前，uber内部使用的分布式跟踪系统，是采用的twitter公司的解决方案——`Zipkin`。 后来，uber内部想把Zipkin的拉取改为推送架构，就逐渐的形成了自己的分布式跟踪系统，最终演变为产品：jaeger。

## Merckx

uber早期的追踪系统叫做Merckx，它应该是使用了Zipkin解决方案，Merckx采用了拉取架构，可以从kafka队列中拉取数据流。但是它最大的不足之处表现在两个方面：

1. 它的设计主要面向Uber使用整体式API的年代。缺乏分布式上下文传播context的概念，虽然可以记录SQL查询、Redis调用，甚至对其他服务的调用，但是无法进一步深入。(具体不太清楚)
2. 另一个有趣的局限是数据存储在全局线程的本地存储中，随着Tornado web框架的引入，这种方式变得不可行。

## TChannel

随后，随着微服务的需要求到来，RPC框架变得越来越重要，2015年初内部开始开发RPC框架——TChannel。其中设计目标之一是将类似于Dapper分布式追踪能力融入到协议中，而OpenTracing标准产生于2016年11月份左右，所以TChannel一开始并不遵循OpenTracing标准。至于后面的发展，后面再看。::TODO

虽然TChannel是与Zipkin解决方案完全无关，但是还是借鉴了后者的一些追踪设计。从内部来看，TChannel的Span在格式上与Zipkin几乎完全相同，也使用了Zipkin所定义的注释，例如："cs"(Client Send)和"cr"(Client Receive)。

TChannel使用追踪报告程序（Reporter）接口将收集到的进程外追踪Span发送至追踪系统的后端。该技术自带的库默认包含一个使用TChannel本身和Hyperbahn实现的报告程序以及发现和路由层，借此将Thrift格式的Span发送至收集器集群。

TChannel客户端库接近我们所需要的分布式追踪系统，该客户端库提供了下列模块：

1. 追踪上下文传播以及带内请求（IPC/RPC）
2. 通过编排API记录追踪Span；（也就是，把trace跟踪进行组件封装）
3. 追踪上下文的进程内传播；（这个一般都很少使用）
4. 将进程外追踪数据报告至追踪后端所需的格式和机制（也就是，Span相关数据转换为Collector和Stoage能够接收和存储的数据格式，这个工作既可以交给Collector来做，也可以Agent主动做好再推送）

该系统唯独缺少了追踪后端本身，即Collector和Storage。追踪上下文的传输格式和报表程序使用的默认Thrift格式在设计上都可以非常简单直接地将TChannel和Zipkin后端集成。然而当时只能通过Scribe将Span发送至Zipkin，而Zipkin只支持Cassandra格式的数据存储。因为当时Uber对这个存储没有什么技术经验，所以，他们自己开发了一套后端原型系统，并结合Zipkin UI的一些自定义组件构建了一个完成的分布式跟踪系统。

也即，uber开发的后端原型系统Collector和Storage分别是tcollector(node.js)和Riak存储(Spans)，Solr索引库(Indexing), 然后通过Zipkin UI来进行查询。

但是随着业务迅猛发展, 后端原型系统架构所使用的Riak/Solr存储系统无法妥善缩放以适应Uber的流量，同时很多查询功能依然无法与Zipkin UI实现足够好的交互操作。同时Uber内部系统语言上的异构，以及还有很多核心业务使用的自己RPC框架，这些异构的技术环境使得分布式追踪系统的构建变得困难。

## Jaeger

针对Merckx产品和Uber内部技术的异构，使得需要更专职的团队做分布式跟踪系统——Jaeger。Jaeger的目标：**将现有的Merckx原型系统转换为可以全局运用的生产系统，让分布式追踪功能可以适用并适应Uber的微服务**

新的团队在Cassandra集群方面已经具备运维经验，该数据库直接为Zipkin后端提供支持，因此团队决定弃用Riak/Solr存储系统。同时，为了接受TChannel流量并将数据兼容Zipkin的二进制格式存储在Cassandra中，使用Golang重新实现了collector。这样对于Zipkin的dashboard就无需改动，完全兼容了。

同时一个很大的改进点，他们还为每个收集器构建了一套可动态配置的倍增系统(Multiplication factor)，借此将入站流量倍增N次，这主要是为了分布式跟踪系统的压测，看看Jaeger的延展性。

这里我们可以看到，Jaeger的早期架构依然依赖于Zipkin UI和Zipkin存储格式。

还面临的一个业务需求，uber内部还有很多核心业务没有使用TChannel RPC框架，为了给公司内部提供透明无侵入的trace服务，组件或者公共服务的编排变得非常重要，各种语言的客户端库提供，为了用不同语言提供一致的编排API，所有客户端库从一开始就采用了**OpenTracing API**。

Jaeger还提供了一个采样策略，防止流量过大，trace对业务造成抖动比较大。策略包括：**1. 全量采样；2. 基于概率的采样；3.限速采样**


Jaeger将有关最恰当的采样策略决策交给追踪后端系统Collector服务，服务的开发者不再需要猜测最合适的采样速率。而后端可以根据流量变化动态地调整采样速率。

```shell
ps: 其实这里也有另一个问题：如果交给后端collector服务进行采样决策，那么agent肯定是全量trace推送给collector，那么agent所在的业务服务压力也大，同时TChannel网络的压力也很大

对于上面我提到的一个问题，Jaeger是这样回答的：

	后端collector服务可以按照流量模式的变化动态地调整采样速率，并反馈到各个服务的agent，形成反馈环路, 在线路中叫做：Control Flow
	
上面的回答，非常吸引人；因为它既不需要业务方考虑流量的增长趋势来选择合适的采样速率，同样，Jaeger的采样率是基于全局流量控制的，所以它具有动态的调整和反馈。这样的策略让人非常舒适
```

另一个需要解决的问题是，TChannel框架的服务发现和服务注册，需要依赖Hyperbahn。但是对于希望在自己的服务中运用追踪能力的工程师，这种依赖造成了不必要的摩擦。(ps: 这句话含义不是很明白？是有自己的服务发现和服务注册吗？比如：etcd，zookeeper, consul等)


为了解决上面这个问题，我们事先了一种jaeger-agent边车(Sidecar)进程, 并将其作为基础架构组件，与负责收集度量值的代理一起部署到所有宿主机上，所有与路由与发现有关的依赖项都封装在这个jaeger-agent中。

此外uber还重新设计了客户端库，可将追踪Span报告给本地UDP端口，并能轮询本地会换接口上的代理获取采样策略。新的客户端只需要最基本的网络库，架构上的这种变化向着我们先追踪后采样的愿景迈出了一大步，我们可以在代理的内存中对追踪记录进行缓存，**这点类似于Appdash的ChunkedCollector方法，在agent以时间和大小两个维度进行缓存。**


目前的Jaeger架构：后端组件使用Golang实现，客户端库使用了四种支持OpenTracing标准的语言，一个机遇React的Web前端，以及一个机遇Apache Spark的后处理和聚合数据管道。

## Jaeger UI

Zipkin UI是Uber在Jaeger中使用的最后一个第三方软件。由于要将Span以Zipkin Thrift格式存储在Cassandra中并与UI兼容，这对后端和数据模型都有了很大的限制，而且数据转换也是个频繁的操作。尤其是Zipkin模型不支持OpenTracing标准；不支持客户端库两个非常重要的功能：1. 键值对日志；2. 更为通用的有向无环图而非span树所代表的跟踪。所以uber下决心彻底革新后端所用的数据模型，并编写新的UI。则表示Collector和Storage的数据存储模型抛弃了Zipkin，使用了OpenTracing标准。其他优化点这里不展开了。


## 参考资料

[sidecar](https://docs.microsoft.com/zh-cn/azure/architecture/patterns/sidecar)

[优步分布式追踪技术再度精进](http://www.infoq.com/cn/articles/evolving-distributed-tracing-at-uber-engineering#anch150996)


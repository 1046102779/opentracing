# TChannel流量指标

本文档定义了每种实现语言中TChannel stack必须发出的通用流量指标。度量指标定义为名称（必需的和可选的tags），其中包含有关特定事件的其他上下文。不支持tags(例如：stasd或者carbon)的度量指标收集系统，应将这个tag信息转换为度量标准名称层次结构中的组件。

在这篇文章中的这个指标对任意应用程序都是通用的；他们不包括在Hyperbahn路由层的特殊指标。

### Relationship to Zipkin

Zipkin跟踪信息可用于导出包含在指标中的时间信息， 但是这些指标不能包含span特殊信息。在某种程度上，可以将定时度量视为在该过程中收集的特定于请求的定时信息的预聚合。


### Relationship to Circuit Breaking

应该使用相同的源事件来发出用于graphql/monitoring的度量指标，并收集熔断器的统计数据，但这两个是独立的系统，并且应该具有单独的存储和数据循环模型


## Common Tags

所有指标必须包含以下tags：

1. `app`。 客户端或者服务端的应用名称/ID。TChannel指标按照层次结构排列——进程运行一个或者多个应用程序，应用程序托管一个或者多个服务， 服务暴露一个或者多个endpoints，一个应用程序可以在多个进程中传播。应用程序名称被认为是TChannel stack的已知先验，并且不依赖于来自线路的任何信息。TChannel实现应该提供一种方法将应用程序名称传递给它们暴露的任何初始化函数。


所有指标可以包括以下tags：

1. `host`。运行报告者的host名称；
2. `cluster`。 运行报告者的集群身份ID；

区分节点运行在canary vs 生产环境 vs 私有模式

3. `version`。 正在运行的应用程序版本号。 这个版本号作为指标由进程发送给Zipkin的Annotations中存储，以支持相关指标数据与依赖性调用图关联起来。


## Call Metrics

call metrics跟踪有关应用程序之间的RPC调用数据统计。

### Outbound call metrics

outbound call metrics是指作为client调用server的rpc指标。call定义为可见的应用程序调用代码，除非下面特别提到，否则speculative请求和重试不会包含在outbound call metrics中。

所有outbound call metrics必须包含`service`和`target-service`， 并且应该包含`target-endpoint` tag。这个`service` tag是表示这个发起这次rpc调用的服务名(transport headers中的key是`cn`, caller name), 这个`target-service` tag是包含被调服务名(在"call req"消息格式中的service字段表示)， 这个`target-endpoint` tag是包含目标服务endpoint的id(在消息由arg1表示)。如果这个endpoint值空间是有限的，这些实现应该仅仅包括目标endpoint；"as" transport，表示"arg schema"， 例如：值为"HTTP-over-tchannel", 在他们的endpoints中包含了uuid，使得这个值空间的利用率最大化。这些实现应该取消target-endpoint tag或者将arg1值标准化为有限集。

#### Counters

**outbound.calls.send**

这个服务调用目标服务的总次数

**outbound.calls.success**

这个服务调用目标服务，并且响应成功的总次数

**outbound.calls.system-errors**

这个服务调用目标服务，并报告给服务的错误帧响应总次数

**outbound.calls.per-attempt.system-errors**

这个服务调用目标服务，在每次发起调用或者调用失败触发重试时，错误帧响应的总次数

**outbound.calls.operational-errors**

这个服务调用目标服务发送"call init"时发送给应用程序错误的总次数。 这个不是错误帧，实际上是本地socket级别错误或者超时错误。应该包含一个`type` tag执行接收的错误类型。标准化操作错误类型是在核心指标之外

**outbound.calls.per-attempt.operational-errors**

这个服务调用目标服务每次尝试调用发生的错误总次数，这个不是错误帧，除了本地socket错误和超时错误。应该包括一个error的`type`tag。标准化操作错误类型是在核心指标之外

**outbound.calls.app-errors** 

这个服务调用目标服务，响应给应用程序CallResponse/NotOk的总数量。可以包含一个应用程序业务级别的错误`type` tag。标准化应用业务级别错误在核心指标之外。

**outbound.calls.per-attempt.app-errors**

这个服务调用目标服务，每次尝试发生的错误总次数。可以包含一个应用业务级别错误的`type` tag。标准化应用程序业务级别错误在核心指标之外。

**outbound.calls.retries**

这个服务发起一个调用，内部所做的重试次数。应该有一个每次请求重试次数的指标`retry-count` tag，例如，一次请求所调用的重试次数为1，则`retry-count` tag值为1等。

**outbound.request.size**

所有发送到目标服务的总字节数累计和，包括帧信息部分。

**outbound.response.size**

由目标服务响应，且没有框架内的系统错误，所累积的总字节数和，包括帧信息部分。


#### Timers

**outbound.calls.latency**

这个服务调用目标服务所发生的端到端延迟（单位：ms）。这个是一次应用程序调用到获取结果所测量的时间。它的测量方式是，从应用发起请求到接收到最后一帧并处理完成所消耗的时间，这包括所有的重试次数。

**outbound.calls.per-attempt.latency**

尝试一次调用(初始化请求或者重试)所耗费的时间。测试方式是，从第一个请求写入到帧，到最后一个响应帧接收所耗费的时间。这次调用应该包含两端的host:port tags, 重试次数的tag同上。

### Inbound call metrics

流入调用指标，也称接收来自外部服务的调用。Inbound call metrics应该包含`serivce`, `endpoint`和`calling-service` tags。这个service和endpoint tag只想被调用服务的服务名和端口号，这个calling-service tag是指发起这次请求调用的服务名称。

#### Counters

**inbound.calls.recvd**

这个服务接收调用的总次数

**inbound.calls.success**

这个服务接收调用，并且返回成功的总次数

**inbound.calls.system-errors**

这个服务接收调用，并错误响应的总次数。必须包含一个类型为error的tag指标，其值表示 outbound.calls.system-errors错误

**inbound.calls.app-errors**

这个服务接收调用，并响应CallResponse/NotOk的总次数。可以包含一个应用程序业务级别的错误`type`tag

**inbound.cancels.requested**

这个服务接收并取消请求的总次数。

**inbound.cancels.honored**

这个服务所取消的取消总次数。在分发到一个应用程序之前，造成调用被丢弃，则表示一个cancel被取消了。另一种是，如果这个应用程序丢弃了这个请求调用，也表示一个cancel被取消了。

**inbound.protocol-errors**

被调用者反馈给调用者的协议错误消息总次数

**inbound.request.size**

被调用者接收到所有请求数据的总字节数，包括帧信息。

**inbound.response.size**

被调用者响应所有请求，且没有发生报错的数据总字节数，包括帧信息

#### Timers

**inbound.calls.latency**

被掉服务处理请求所耗费的时间，单位：ms。它是指接收外部请求开始，到处理请求结束并返回最后一帧所耗费的时间.

### Connection Metrics

Connection指标测量统计数与socket connections相关。所有的connection指标必须包含`host-port`和`peer-host-port` tags. 这个`host-port` tag是指reporting进程的host和端口号。对于短连接客户端，它可以是0.0.0.0:0。这个`peer-host-port` tag是对等一方的host和端口号。

#### Gauges

对于Gauges概念，我们在Prometheus了解过，它不是线性递增的。

**connections.active**

这个进程与被调用者建立的当前活跃连接数。active connection是用来初始化或者接收流量的连接，也包括以一个友好地shutdown的连接

#### Counters

**connections.initiated**

在进程的整个生命周期内初始化建立连接到目标服务的所有连接总数

**connections.connect-errors**

当这个进程尝试初始化一个连接到被调服务时的错误总次数

**connections.accepted**

在这个进程的生命周期内，目标服务接受连接的总次数。

**connections.accept-errors**

在这个进程的生命周期内，接受连接错误的总次数

**connections.errors**

在连接建立初始化时，连接发生错误的总次数。必须包括一个指向错误类型的`type`tag标签。标准的type tags包括：

1. *network-error*。 这个连接遭遇到网络错误(例如：ECONNRESET)。
2. *network-timeout*。在read/write期间，这个连接接收到的网络超时。
3. *protocol-error*。被调用服务响应一个违法协议消息。
4. *peer-protocol-error*。这个进程发送一个消息，被调用服务发送一个解析协议错误的消息。


**connections.closed**

由于error或者友好地shutdown，连接关闭的总数量（例如，空闲连接超时，应用初始化关闭）。可以包含一个指向关闭原因的`reason` tag。标准的reason tags包括：

1. *idle-timeout*。由于连接空闲超时导致连接被关闭。
2. *app-initated*。 由于应用初始化连接导致的连接关闭；
3. *protocol-error*。由于协议错误导致连接关闭。
4. *network-error*。由于网络层错误导致连接关闭。
5. *network-timeout*。由于网络超时导致连接关闭。
6. *peer-closed*。有关对方引起的连接关闭


**connections.bytes-sent**

在已建立的所有连接上，所有发送的流量字节总数。

**connections.bytes-recvd**

在已建立的所有连接上，所有接收的流量字节总数

### Circuit Metrics

Circuit状态指标是由Hyperbahn服务代理发送的，不是由TChannel直接发送的。这个指标是从circuits实例中的`circuitStateChange`事件获取的

Circuit状态变化，健康或者不健康, 并在每个Circuit中跟踪统计数据。代替对tag的支持，Circuits是有调用者名称和服务名称的临时索引。

这个Circuit技术统计方式模式如下：

1. `circuits`
2. `{healthy, unhealthy}`，并且包含`total`， 或者`by-caller.$caller.$service.$endpoint`， 或者 `by-service.$service.$caller.$endpoint`中的一种。


### Rate Limiting Metris

这个流控指标跟踪了每秒发送到一个服务的请求数(RPS)， 以及由于流控生效而产生的busy响应。

#### Gauges

**rate-limiting.service-rps**

每秒发送给一个服务的请求数。它必须包含`target-service` tag。

**rate-limiting.service-rps-limit**

每秒发送给一个服务的最大请求数量。如果这个服务的RPS达到了这个上限，请求将会返回busy帧的响应消息。它必须包含一个`target-service` tag。

**rate-limiting.total-rps**

一个tchannel进程接收的请求数量

**rate-limiting.total-rps-limit**

一个tchannel进程能够处理的最大请求数。如果这个RPS达到了这个上限，请求将会获得busy帧的响应消息。

#### Counters

**rate-limiting.service-busy**

由于发送给一个服务的请求数量超过了这个限制，生成busy帧响应的数量。它必须包括`target-service` tag。

**rate-limiting.total-busy**

由于发送给一个tchannel进程的请求数量超过了这个限制，生成busy帧响应的总数量。当这个信息是可用时，它应该包含`target-service` tag。

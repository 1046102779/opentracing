# 创建一个Go+Thrift+Hyperbahn服务

符合这个指南的代码[示例](https://github.com/uber/tchannel-go/blob/dev/examples/keyvalue)

对于Go语言，TChannel+Thrift集成使用thrift-gen生成的代码。

## 依赖

在follow这个指南之前，确保你的GOPATH路径是已经设置了。

你需要`go get`遵循：

- github.com/uber/tchannel-go
- github.com/uber/tchannel-go/hyperbahn
- github.com/uber/tchannel-go/thrift
- github.com/uber/tchannel-go/thrift/thrift-gen


使用[Godep](https://github.com/tools/godep)去管理依赖包，这个API仍然在开发中。

这个例子假定，这个服务在下面的目录下创建：`$GOPATH/src/github.com/uber/tchannel-go/examples/keyvalue`

你应该使用你自己的path，并根据自身情况更新引入包

## Thrift服务

创建[thrift](https://thrift.apache.org/)文件去定义你的服务。对于这个指南，我们使用：

`keyvalue.thrift`:

```shell
service baseService {
	string HealthCheck()
}

exception KeyNotFound {
	1: string key
}

exception InvalidKey {}

service KeyValue extends baseService {
	string Get(1: string key) throws (
		1: KeyNotFound notfound
		2: InvalidKey invalidKey
	)
	
	void Set(1: strign key, 2: string value)
}

exception NotAuthorized {}

service Admin extends baseService {
	void clearAll() throws (
		1: NotAuthorized notAuthorized
	)
}
```

这个Thrift规范定义了两个服务：

1. `KeyValue`：一个简单的key-value存储；
2. `Admin`: 对于key-value存储的管理

这两个服务继承了`baseService`，也就继承了`HealthCheck`方法。

这些方法可能返回异常，而不是返回预期的结果，这些异常也在规范中定义。

一旦你定义了你的服务，应该通过客户库运行以下命令，去生成Thrift服务：

```shell
cd $GOPATH/src/github.com/tchannel-go/examples/keyvalue

thrift-gen --generateThrift --inputFile keyvalue.thrift
```

这个运行Thrift编译器，然后生成service和client绑定。你能够手动运行这些命令：

```shell
thrift -r --gen go:thrift_import=github.com/apache/thrift/lib/go/thrift keyvalue.thrift

thrift-gen --inputFile "$THRIFTFILE" --outputFile "THRIFT_FILE_FOLDER/gen-go/thriftName/tchan-keyvalue.go"
```

## Go server

为了使server ready，下面是需要做的：

1. 在网络协议层创建TChannel；
2. 创建一个handler，来处理在Thrift文件中定义的方法，然后把它注册到tchannel/thrift；
3. 创建一个Hyperbahn client和发布你的服务到Hyperbahn。

### Create a TChannel

使用[tchannel.NewChannel](http://godoc.org/github.com/uber/tchannel-go#NewChannel)创建一个channel，使用[Channel.ListenAndServe](http://godoc.org/github.com/uber/tchannel-go#Channel.ListenAndServe)

监听的地址应该是一个远程IP，它能够用于其他机器的请求连接建立。你能够有使用[tchannel.ListenIP](http://godoc.org/github.com/uber/tchannel-go#ListenIP), 它使用探索式方式选择一个好的服务

当创建一个channel时，你能够传入[options](http://godoc.org/github.com/uber/tchannel-go#ChannelOptions)参数

### Create and register Thrift handler

创建一个由Thrift生成接口的自定义方法。你能通过在`gen-go/keyvalue/thcan-keyvalue.go`文件查找这个接口。例如，生成的文件中的接口如下：

```shell
type TChannelAdmin interface{
	HealthCheck(ctx thrift.Context) (string, error)
	ClearAll(ctx thrift.Context) error
}

type TChannelKeyValue interface{
	Get(ctx thrift.Context, key string) (string, error)
	HealthCheck(ctx thrift.Context) (string, error)
	Set(ctx thrift.Context, key string, value string) error
}
```

创建一个handler类型的实例，然后创建一个[thrift.Server](http://godoc.org/github.com/uber/tchannel-go/thrift#NewServer) 和[register](http://godoc.org/github.com/uber/tchannel-go/thrift#Server.Register)你的thrift handler。 你能够在相同的`thrift.Server`上注册多Thrift服务


每个Handler方法是运行在一个新的goroutine，因此必须是线程安全的。你的handler方法能够返回两类错误：

- 在Thrift文件中声明的Error(例如：`KeyNotFound`)
- Unexpected errors

如果你返回了一个unexpected error，一个error帧发送带有Thrift的消息。如果能够枚举错误类型，最好在Thrift文件中声明他们，然后直接返回。例如：

```shell
if value, ok := map[key]; ok {
	return value, ""
}

return "", &keyvalue.KeyNotFound{Key: key}
```

### Advertise with Hyperbahn

使用[hyperbahn.NewClient](http://godoc.org/github.com/uber/tchannel-go/hyperbahn#NewClient)创建一个Hyperbahn client，在当前环境下它要求冲一个配置文件中加载一个Hyperbahn配置对象。当创建client时，你也能够传递更多的[options](http://godoc.org/github.com/uber/tchannel-go/hyperbahn#ClientOptions)

调用[Advertise](http://godoc.org/github.com/uber/tchannel-go/hyperbahn#Client.Advertise)去在Hyperbahn发布这个服务。

### Serving

你的服务现在已经在Hyperbahn上了。你能够使用[tcurl](https://github.com/uber/tcurl)进行调用测试：

```shell
node tcurl.js -p [HYPERBAHN-HOSTPORT] -t [DIR-TO-THRIFT] keyvalue KeyValue::Set -3 '{"key": "hello", "value": "world"}'

node tcurl.js -p [HYPERBAHN-HOSTPORT] -t [DIR-TO-THRIFT] keyvalue KeyValue::Get -3 '{"key": "hello"}'
```

用一个Hyperbahn节点的Host:port替代`[HYPERBAHN-HOSTPORT]`， .thrift文件存储在`[DIR-TO-THRIFT]`目录下

你的服务现在能够用任何语言通过Hyperbahn+TChannel进行访问

## Go client

注意：这个client实现仍然在积极的开发中

为了使一个client能够通信，你需要做：

1. 创建一个TChannel(or 重用一个已存在的TChannel)
2. 创建Hyperbahn
3. 创建一个Thrift+TChannel client；
4. 使用Thrift client做远程调用

### Create a TChannel

TChannel是一个双向流，因此这个client使用与server相同的代码(tchannel.NewChannel)去创建一个TChannel。你不需要在channel上调用ListenAndServe。甚至这个channel不需要一个host service，但是对于TChannel，一个服务名是需要的。这个serviceName对于这个client应该是唯一的。

你能够使用一个已存在的TChannel发起client调用。

### Set up Hyperbahn

与server代码类似，使用Hyperbahn.NewClient去创建一个新的Hyperbahn client。你不需要调用Advertise，这个client没有任何服务需要发布到Hyperbahn中

如果你已经创建一个已存在的client，你不需要在做任何事情了。

### Create a Thrift client

这个Thrift client有两部分：

1. 这个`thrift.TChanClient`配置为命中特定的Hyperbahn服务。
2. 使用底层的`thrift.TChannClient`生成的client，为一个指定thrift服务调用方法。


创建一个`thrift.TChanClient`, 使用`thrift.NewClient`。这个client能够用于创建一个生成的client：

```shell
thriftClient := thrift.NewClient(ch, "keyvalue", nil)

client := keyvalue.NewTChanKeyValueClient(thriftClient)

adminClient := keyvalue.NewTChanAdminClient(thriftClient)
```

### Make remote calls

在这个client上通过TChannel发起一个远程过程调用，例如：

```shell
err := client.Get(ctx, "hello", "world")

val, err := client.Get(ctx, "hello")
```

当发起方法调用时，你必须传入一个context。这个调用会携带deadline、tracing和应用headers。一个简单的root context是：

```shell
ctx, cancel := thrift.NewContext(time.Second)
```

所有通过TChannel的调用必须要求有timeout，tracing信息。NewContext应该仅仅用于调用链的开头。所有其他节点通过传入的context，并传递下去。当你通过context传递，你也会传递deadline，tracing信息和这个headers。trace context的上下文传播机制。

## Headers

Thrift+TChannel允许client发送headers， servers能够增加响应headers到任何响应中；

在Go语言中，在使用[WithHeaders](http://godoc.org/github.com/uber/tchannel-go/thrift#WithHeaders)调用之前，headers附加在context上。

```shell
headers := map[string]string{"user":"prashant"}

ctx, cancel := thrift.NewContext(time.Second)

ctx = thrift.WithHeaders(ctx)
```

这个server通过[Headers](http://godoc.org/github.com/uber/tchannel-go/thrift#Context)读取这些headers，并且能够通过`SetResponseHeaders`方法，增加额外的响应头。

```shell
func (h *kvHandler) ClearAll(ctx thrift.Context) {
	headers := ctx.Headers()
	
	respHeaders := map[string]string{
		"count": 10,
	}	
	ctx.SetResponseHeaders(respHeaders)
}
```

在同一个context，通过调用`ctx.REsponseHeaders`方法，这个client能够读取到响应头。

```shell
ctx := thrift.WithHeaders(thrift.NewContext(time.Second), headers)
err := adminClient.ClearAll()

responseHeaders := ctx.ResponseHeaders()
```

头部不要加上调用方法的参数，应该放入Thrift request/response结构相应位置。


## Limitations & Upcoming Changes

TChannel的对等选择还没有针对节点的详细运行状况模型，并且选择不会平衡节点之间的负载。

这个thrift-gen自动生成的代码是新的，可能不支持所有Thrift功能(例如：注释，includes，多文件等)

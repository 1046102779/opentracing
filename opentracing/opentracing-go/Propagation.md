# Propagation

1. 跨进程的trace信息传输通过SpanContext传递baggage信息，把baggage存储到context上下文中，则需要key:value, 这个key决定了value值和value类型；
2. key:value存储到context中，需要借助于读写操作；`notice: 这个借鉴了io.Reader和io.Writer等思想，通过组合模式使得实现变得更加灵活`
3. 目前支持三种key值，对应三种value值类型，分别是：byte流、TextMap和http.Header；其中后两者都是map结构；只是RPC或者Web时，用http.Header，其他使用TextMap或者byte流；

## Baggage读写

目前支持三种类型的value，BuiltinFormat={Binary=0, TextMap=1, HTTPHeaders=2}, 对于第一种比特流方式，直接通过`io.Reader`和`io.Writer`方式即可；

对于后两者的读写操作：

```shell
type TextMapWriter interface {
	Set(key, val string)
}

type TextMapReader interface {
	ForeachKey(handler func(key, value string) error) error
}
```

从TextMapReader来看，ForeachKey的传参类型是一个函数，使得读取交给了具体的trace系统实现，灵活度高；

Baggage的TextMap和Http.Header两种存储，分别实现了上面两个接口：

```shell
// TextMap
type TextMapCarrier map[string]string

func (c TextMapCarrier) ForeachKey(handler func(key, val string) error) error{
	for k, v := range c{
		if err := handler(k, v); err!=nil{
			return err
		}
	}
	return nil
}

func (c TextMapCarrier) Set(key, val string){
	c[key] = val
}


// Http.Header
type HTTPHeadersCarrier http.Header

func (c HTTPHeadersCarrier) Set(key, val string){
	h:= http.Header(c)
	h.Add(key, val)
}

func (c HTTPHeadersCarrier) ForeachKey(handler func(key, value string) error) error{
	for k, vals:= range c{
		for _, v := range vals{
			if err:= handler(k, v); err !=nil{
				return errr
			}
		}
	}
	return nil
}
```

# ext/tags

根据OpenTracing标准，[《OpenTracing APIs》](https://gocn.vip/article/858)中的数据约定模块，已经在标准中集成了一些常用的Span Tag。在opentracing-go库中，也就集成了这一部分子集，如：

```shell
SpanKind："span.kind" 代表有关RPC、WEB、消息发布与订阅等关系；这类Span Tag值={"Client", "Server", "Producer", "Consumer"}

第三方库或者组件 Component: "component"; 这类span tag值表示调用的第三方库或者组件名称；

采样率：SamplingPriority: "sampling.priority"; 这类span tag值大于或者等于0，当value>0时，尽量保留这条trace；否则，丢弃这条trace；

peer：这类span tag表示对等的相关信息，也即调用方上游或者下游的相关服务信息；具体有：
PeerService: "peer.service"; tag值为服务名；
PeerAddress: "peer.address"; tag值为服务连接地址；
PeerHostName："peer.hostname"; tag值为主机名；
PeerHostIPv4, PeerHostIPv6: "peer.ipv4"和"peer.ipv6"; tag值为IP地址
PeerPort: "peer.port"；tag值为服务端口；

HTTP相关tags列表：
HTTPUrl: "http.url"; tag值为http url；
HTTPMethod："http.method"; tag值为http.method={"get", "post", "delete", "head", ...};
HTTPStatusCode: "http.status_code"; tag值为http.status_code={403, 404, 200, 502, ...};

DB相关tags列表：
DBInstance: "db.instance", tag值为db实例名称；
DBStatement："db.statement", tag值为db sql语句；
DBType： "db.type", tag值为db type = {"redis", "mysql", "progresql", ...};
DBUser: "db.user", tag值为访问db的用户名
// 这里不给出DBPassword是因为，passwd属于非常敏感信息；

// message bus 消息总线tag：
MessageBusDestination: "message_bus.detination", tag值为address；
消息总线，我还不太了解；

// error tag: 表示span执行单元过程是否有业务系统发生错误，true|false
Error: "error", tag值：true|false；
```

**我们注意到，当跟踪某一类tag A时，如果这类tag存在多个指标；这A.a, A.b, ..., A.z等方式是非常友好的，便于识别含义，且不会冲突；**

上面的这些tag都会有相应的读写操作：Set, 但是**目前没有发现Tag读取操作**, 大多数tag set操作类似于：

```shell
func (tag xxxTagName) Set(span opentracing.Span, value xxtype){
	span.Set(string(tag), value)
}
```

**这里说明一点:  对于已知的TagName和可枚举的value，直接使用StartSpanOptions中的Apply方法即可，如：span.kind； 其他的通过TagName类型自带的Set实现存储，这些tag数据都是存储在StartSpanOptions的tags中**

# 参考资料

[opentracing-go](https://github.com/opentracing/opentracing-go)
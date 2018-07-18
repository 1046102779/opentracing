# TChannel与HTTP的错误码映射

**稳定性：不稳定**

当发送请求和响应请求时TChannel能够返回不同类型的错误

## TChannel Client Errors

作为http状态机集成的一部分，tchannel错误需要映射到http合适的错误码，以方便http客户端能够区分不同类型的错误并作出合适的后续动作。

tchannel错误码在[Jaeger TChannel ——protocol](https://gocn.vip/article/900)一文中已经陈述过了

http的状态返回码: [http status code](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)

下面是tchannel与http的状态错误码映射关系表：

| code | name | http status code |
| --- | --- | --- |
| `0x01` | timeout | 504 Gateway Timeout |
| `0x02` | cancelled | 500 TChannel Cancelled |
| `0x03` | busy | 429 Too Many Requests |
| `0x04` | declined | 503 Service Unavailable |
| `0x05` | upexpected error | 500 Internal Server Error |
| `0x06` | bad request | 400 Bad Request |
| `0x07` | network error | 500 TChannel Network Error |
| `0x08` | unhealthy | 503 | Service Unvailable |
| `0xFF` | fatal protocol error | 500 TChannel Protocol Error |


# HTTP over TChannel

这篇文章阐述了我们怎么样把HTTP编码进TChannel。

对于HTTP请求调用，这个Transport Headers中存在key：`as`， 值必须设置为`http`。请求时消息类型为"call req"和响应时消息类型为"call res"，它们带有`arg1`、`arg2`和`arg3`，定义如下：

## Arguments

1. `arg1` 是一个任意circuit字符串，可以留空；
2. `arg2` 是一个编码后的request/response元数据，详见下文；
3. `arg3` 是来自http请求/响应的原始字节流

### `arg2`：request meta data

Binary schema:

```shell
method~1
url~N
numHeaders: 2(headerName~2 headerValue~2){numHeaders}

例如：
method: GET
url: /v1/accounts/:account_id
numHeaders:{secret: xxxx2j02}
```

注意：

1. 这个url字段长度是变长的。`则只能根据arg2的总长度，间接计算url的变长大小`

### `arg2`：response meta data

Binary schema

```shell
statusCode:2
message~N
numHeaders:2(headerName~2 headerValue~2){numHeaders}

例如：
statusCode: 200
message: {err_code: 0, err_msg: ""}
numHeaders: {status: "00"}
```

注意：

1. statusCode是HTTP的状态响应码；
2. 消息是utf-8编码；
3. 消息长度也是一个变长；
4. headers可以使用multi-map或者列表键值对实现`这个在Appdash的RawSpan中见过`；单值是不够的

# JSON over TChannel

这篇文档阐述了我们怎么样把JSON编码进TChannel中

对于JSON的请求调用，则Transport Headers一定有key: `as`，值为`json`的键值对。对于Request消息类型为"call req"，和Response消息类型为"call res", 都会带有`arg1`，`arg2`和`arg3`，这三个参数的定义如下：

对于每个"call req"消息，这个服务名应该设置为被调用的TChannel服务

对于每个"call res"消息，如果这个响应是成功的，这个响应码必须设置为`0`；如果这个响应是失败的，则这个响应码必须设置为`1`。

## Arguments

对于"call req"和"call res"：

1. `arg1`一定是方法名；
2. `arg2`一定是JSON编码的application headers
3. `arg3`一定是application response


### arg1

这个方法一定是UTF-8编码。建议您使用字母数字字符和`_`。

### arg3

`arg3`必须是JSON序列化编码

对于“call req”消息，这只是一个任意的JSON有效载荷

对于"call req"消息：

1. 在成功的情况下，响应是任意JSON负载数据；
2. 在失败的情况下，响应时JSON错误信息。errors中有个`message`字段，以及一个`type`字段(告知错误类型)

# TChannel Service

客户端库应该注册一个默认服务，这个服务提供有关客户端库和内部状态的元数据信息。此默认服务应该在"tchannel"下注册，可在不知道托管endpoint应用程序的服务名称情况下使用该服务

这个Thrift scheme详见[meta.thrift](https://github.com/uber/tchannel/blob/master/thrift/meta.thrift)

# 多语言支持

## TChannel

TChannel是一个支持多路复用和帧协议的RPC服务框架

这个协议当前已经支持的语言，如下所示

1. Go
	
	1). [Guide](https://github.com/uber/tchannel/blob/master/docs/go-guide.md)
	
	2). [API Documentation](http://godoc.org/github.com/uber/tchannel-go)
2. Node
	
	1). [Guide](http://tchannel-node.readthedocs.org/en/latest/GUIDE/)
	
	2). [API Documentation](http://tchannel-node.readthedocs.org/en/latest/)
3. Python
	
	1). [Guide](http://tchannel.readthedocs.org/projects/tchannel-python/en/latest/guide.html)
	
	2). [API Documentation](http://tchannel.readthedocs.org/projects/tchannel-python)
	


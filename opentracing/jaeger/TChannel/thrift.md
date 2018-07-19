# Thrift over TChannel

这篇文章阐述了我们意图在TChannel使用Thrift协议

对于Thrift数据流通过TChannel，则Transport Headers中存在`as=thrift`。请求带有"call req"消息和响应带有"call res"， `arg1, arg2, arg3`值定义在[Arguments](https://github.com/uber/tchannel/blob/master/docs/thrift.md#arguments)。

对于`call res`，

1. 调用成功，则响应码(`code:1`)一定设置为`0x00`。
2. 调用失败，则响应吗(`code:1`)一定设置为`0x01`。

## Arguments

对于 `call req` 和 `call res`,

1. `arg1` 必须设置为请求的方法名；
2. `arg2` 必须是application headers, 格式`nh:2 (k~2 v~2){nh}`
3. `arg3` 必须是Thrift payload

### `arg1`

这个一定是服务方法名和Thrift服务名的级联，通过`::`连接。它与指向服务endpoint是一样的。

例如：对于下面服务的`ping`方法，`arg1`是`PingService::ping`。

```shell
service PingService {
	void ping()
}
```

注意，这个Thrift服务名并不需要与TChannel服务名相同。也就是，在`call req/res`消息协议中的`service~1`可以与定义在Thrift IDL的服务名可以不相同。

### `arg3`

`arg3`必须包含一个使用`TBinaryProtocol`编码的Thrift结构

对于`call req`消息，它是一个仅仅包含这个方法参数的struct结构

对于 `call res`消息，

1. 调用成功时，这个响应包含一个字段身份标志位`0`的结构体，它包含了这个方法的返回值。对于方法返回类型为`void`，这个结构体一定是空的；
2. 调用失败时，这个响应包含一个异常字段标志的结构体。用以表示调用失败返回异常值；

例如：

```shell
service CommentService {
	list<Comment> getComments (
		1: EntityId id
		2: i32 offset
		3: i32 limit
	) throws (
		1: InvalidParametersException invalidParameters
		2: EntityDoesNotExist doesNotExist
	)
}
```

对于`getComments(1234, 10, 1000`，对于`call req`的这个`arg3`将包含以下结构体二进制编码的版本：

```shell
{
	1: 1234,
	2: 10,
	3: 100,
}
```

如果这个调用成功，则`call res`的消息体包含以下二进制编码内容：

```shell
{
	0: [
		{ /* comment fields go here */ },
		{ /* comment fields go here */ },
		// ...
	]
}
```

如果这个调用返回失败，则消息体会带有`EntityDoesNotExist`的异常，这个body包含以下二进制编码的结构体：

```shell
{
	2: { /*EntityDoesNotExist fields go here */ }
}
```

## Multiple Services

为了避免疑惑，这些定义在这个部分被使用。

1. **Service** 是指在Thrift IDL中定义为`service`, 也就是说一个服务单独在Thrift IDL定义为一个serivce；
2. **System** 是指定义在Thrift IDL的所有服务。一个系统包含多个不同的服务；

对于一个系统的Thrift IDL可以包含多个Thrift服务，这些服务划分这个系统为多子域，也就是微服务划分。例如：

```shell
service UserService {
	UserId createUser(1: UserDetail details)
	void verifyEmailAddress(1: UserId userId, 2: VerificationToken token)
}

service PostService {
	PostId submitPost(1: UserId userId, 2: PostInfo post)
}
```

有两种方式使用这种多服务系统：

1. 同一台服务器上有多个不同的服务；
2. 在系统中每个服务不同端口或者不同机器上设置单独的server。消费者需要指定这些不同的端口以供clients访问。

在这个部分我聚焦第一种方案，由于第二种方案每个服务存在多个不同的系统中是非常不同的。

提及`arg1`，每个服务方法都由TChannel服务进行注册，格式`{serviceName}::{methodName}`。上面的例子，我们有三个endpoints，分别是`UserService::createUser`, `UserService::verifyEmailAddress`和`PostService::submitPost`。

当发起请求时，调用者必须使用完整的endpoint。例如：

```shell
send(
	{
		service: "UserSerivce", // 这个是TChannel服务名
		endpoint: "PostService::submitPost",
		// ...
	}
)
```

为方便起见，如果TChannel服务名匹配到Thrift服务名，则客户端实现允许`{serviceName}::`前缀省略。例如：

```shell
send({service: "UserService", endpoint: "createUser"})

// 这实现应该翻译成下面这样：
send({Service: "UserService", endpoint: "UserService::createUser"})
```

## Service Inheritance

Thrift支持服务继承概念，例如：

```shell
service BaseService {
	bool isHealthy()
}

service UserService extends BaseService {
	// ...
}

service PostService extends BaseService {
	// ...
}
```

在这个服务继承例子中，我们不想"parent"服务方法在子服务中注册。则我们不想`BaseService::isHealthy`分别在`UserService`和`PostService`中注册。使用extends方式，则继承了公共服务

## Uncaught Exceptions

当没有捕捉到服务异常时，他们没有在Thrift IDL中定义，服务实例应该尝试响应一个TChannel `error`消息错误，错误码`code:1`为`0x05`(unexpected error)

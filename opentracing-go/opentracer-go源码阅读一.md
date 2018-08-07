# Tracer

## Tracer Interface

```shell
type Tracer interface{
	StartSpan(operationName string, opts ...StartSpanOption) Span
	
	Inject(sm SpanContext, format interface{}, carrier interface{}) error
	
	Extract(format interface{}, carrier interface{}) (SpanContext, error)
}
```

以上是实现一个tracer调用链跟踪的**最小子集**，任何一个底层跟踪系统的实现，如果实现了它，则可以把整个调用链串起来。

在Tracer interface中StartSpan方法传参，传入Span结构体参数，如：`SpanReference列表`，`创建时间`和`Span Tag信息`；其中:

`SpanReference`列表：表示新创建的Span与【curr_depth+1】Span之间的关系，如FollowsFrom和ChildOf；注意，这个是RPC/IPC跨进程调用关系，所以SpanContext表示OpenTracing标准中可以携带Baggage信息；

`Tags`: 类型：map[string]interface{}, 生命周期：span执行单元；

因为`SpanReference`, `StartTime`和 `Tags`三个变量为StartSpanOptions参数，所以必须实现StartSpanOption接口中Apply方法。 **这个StartSpanOptions代表Span的必要参数子集**。

## globaltracer

每一个微服务都会有一个globaltracer值， 我们通过SetGlobalTracer方法设置业务系统采用的分布式跟踪系统产品；一个产品会有1个到多个微服务，则每个微服务都需要指定全局Tracer变量值，这一个Tracer就代表一个遵循OpenTracing标准的厂商产品，**在一个产品中，所有微服务的Tracer变量值都必须指向同一个厂商产品，否则，分布式跟踪系统服务生效**。

其中，我们可以通过`GlobalTracer()`方法获取微服务中的Tracer值；

如果我们没有对Tracer进行初始化，则意味着没有指定采用哪个厂商的分布式跟踪系统产品，则默认情况下，采用opentracing-go底层标准下的默认空Tracer——**NoopTracer**，这个是在实现OpenTracing标准API库时，必须要做的，因为如果业务系统采用的组件或者第三方库中有探针，如果不实现空的Tracer，则直接报错，无法使用，从而导致Tracer与业务强耦合了。

## NoopTracer
微服务中默认的空Tracer，包含：NoopTracer、noopSpan和noopSpanContext三个变量值；其中：

1. noopSpanContext是在SpanReference中使用，它代表SpanContext上下文传输tracer时的span获取。并携带Baggage信息；`ForeachBaggageItem(func(k, v string) bool){}`
2. noopSpan实现了Span interface。方法实现全部为空；
3. noopTracer实现了Tracer interface，包括StartSpan、Inject和Extract三个方法；全部为空实现；

微服务中，NoopTracer是Tracer未指定时的默认实现，不会在Tracer中生成任何数据，也不会产生Tracer调用链路；

# Span

## Span interface

```shell
type Span interface{
	// Span执行单元结束
	Finish()
	// 带结束时间和日志记录列表信息，Span执行单元结束；也就是说结束时间可以业务指定; 日志列表也可以直接添加； 目前还不知道这个的使用场景；
	FinishWithOptions(opts FinishOptions)
	// 把Span的Baggage封装成SpanContext
	Context() SpanContext
	// 设置Span的操作名称
	SetOperationName(operationName string) Span
	// 设置Span Tag
	SetTag(key string, value interface{}) Span
	// 设置Span的log.Field列表；
	// span.LogFields(
	// 		log.String("event", "soft error"),
	// )
	LogFields(fields ...log.Field) 
	// key-value列表={"event": "soft error", "type": "cache timeout", "waited.millis":1500}
	LogKV(alternatingKeyValues ...interface{})
	// LogFields与LogKV类似，只是前者已封装好；
	
	// 设置span的Baggage：key-value， 用于跨进程上下文传输
	SetBaggageItem(restrictedKey, value string) Span
	// 通过key获取value；
	BaggageItem(restrictedKey string) string
	// 获取Span所在的调用链tracer
	Tracer() Tracer
	// 废弃，改用LogFields 或者 LogKV
	LogEvent(event string)
	// 同上
	LogEventWithPayload(event string, payload interface{})；
	// 同上
	Log(data LogData)
}
```

## SpanContext

首先，需要看一小段代码:

```shell
package main

import "fmt"

type Fun struct{}

func main() {
    var fun1, fun2 = Fun{}, Fun{}
    if fun1 == fun2 {
        fmt.Println("fun1==fun2")
    } else {
        fmt.Println("fun1!=fun2")
    }
}

// 执行结果：fun1==fun2
// 比较两个值是否相等，取决于：值和类型，都相等这表示相同；
```

Context用于上下文数据传输使用，在OpenTracing标准中，Span之间跨进程调用时，会使用SpanContext传输Baggage携带信息。通过context标准库实现；如：

```shell
type contextKey struct{}

var activeSpanKey = contextKey{}

// 封装span到context中
func ContextWithSpan(ctx context.Context, span Span) context{
	return context.WithValue(ctx, activeSpanKey, span)
}

// 从ctx中通过activeSpanKey取出Span，这里可以看到不同服务的activeSpanKey值，是相同的，上面的DEMO可以说明。
func SpanFromContext(ctx context.Context) Span{
	val:=ctx.Value(activeSpanKey)
	if sp, ok := val.(Span); ok{
		return sp
	}
	return nil
}
```

通过context上下文的activeSpanKey，我们可以获得Span，并创建新的span，如：

```shell
func startSpanFormContextWithTracer(ctx context.Context, tracer Tracer, operationName string, opts ...StartSpanOption) (Span, context.Context){
	// 首先从上下文看是否能够获取到span，如果获取不到，再创建tracer和span；
	if parentSpan:= SpanFromContext(ctx); parentSpan !=nil {
		opts = append(opts, ChildOf(parentSpan.Context()))
	}
	
	span := tracer.StartSpan(operationName, opts...)
	return span, ContextWithSpan(ctx, span)
}
```

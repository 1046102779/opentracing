# Log

该log设计参考了[uber-go/zap](https://github.com/uber-go/zap), 日志值类型非常多，该设计把日志类型分为三类：string, int64和interface{}, 其中：

1. string={string}；
2. int64={bool, int, int32, uint32, int64, uint64, float32, float64}, `notice: 这里没有覆盖int16和uint16`
3. interface{}={error, object, lazyLogger, noop}

根据Span interface中的LogFields和LogKVs方法， log.Field应该会包含{key, value}, 同时因为value值类型包含三类，所以Field的结构如下：

```shell
type Field struct {
	key			string
	fieldType  	fieldType
	numericVal 	int64
	stringVal 	string
	interfaceVal	interface{}
}
```

Field提供了返回key，返回value和String三个方法操作，这已满足需要；

**注意，Span Tags和Span Logs不同的一点在于，Logs是可以累加的，并不是map结构，是列表存储[]log.Fields**

实现Span Logs的标准化结构如下：

```shell
func String(key, val string) Field {
	return Field{
		key: 	 	key,
		fieldType:	stringType,
		stringVal:	val,  
	}
}

// 其他类型都类似处理， 对于error和lazyLogger类型，不需要指定key
func Error(err error) Field{
	return Field{
		key:		"error",
		fieldType:	errorType,
		interfaceVal:	err,
	}
}

// LazyLogger 对厂商暴露出的自定义日志实现
type LazyLogger func(fv Encoder)

func Lazy(ll LazyLogger) Field{
	return Field{
		fieldType: lazyLoggerType,
		interfaceVal: ll,
	}
}

// Noop用于忽略一些日志处理；
// 例如：dev/test/prod的trace日志等
func Noop() Field{
	return Field{
		fieldType: noopType,
	}
}
```

我们可以兑log.Field进行编码操作，类似于json/xml解析库；这个Encoder interface列出了log.Field中value值所有类型；所以，如果要对log.Field编码操作，则需要实现所有值类型，共三大类。


对于Span interface中的LogFields和LogKVs两个方法都是对[]log.Fields进行存储，这两个方法存在一个数据结构化转换方法： **InterleavedKVToFields**, 把key-value键值对列表转换成[]log.Field结构化日志数据。

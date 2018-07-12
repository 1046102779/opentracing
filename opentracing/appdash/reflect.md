我们在[Appdash源码阅读——Annotations与Event](https://gocn.vip/article/881)一节中，了解到Appdash产品的Span信息存储包括两部分SpanID和Annotations，因为Appdash出生比OpenTracing标准早，所以没有遵循后者的标准。Span除了自身信息，其他信息全部是存储在Annotations中，它是slice结构，且里面是key-value存储方式。这里面存放了event事件，为了实现Event和Annotations的数据存储转换，需要提供序列化和反序列化操作。这就引入了reflect。

这节我们来看看Appdash的反射操作。顺便了解了解reflect

校验两个变量是否相等，取决于两部分是否相等。1. 变量类型是否相等；2. 变量值是否相等

# reflect

1. 在把event序列化为Annotations时，如果event没有提供MarshalEvent方法，则就使用默认的序列化方法`flattenValue`。
2. 在把Annotations返回序列化到event时，如果event没有提供UnmarshalEvent方法，则就使用默认的反序列化方法`unflattenValue`.

## flattenValue

```shell
如：e: Event
type Event interface {
	Schema() string // log, msg, timespan Timespan and soon
}

flattenValue("", reflect.ValueOf(e), func(k, v string) {
	as = append(as, Annotation{Key: k, Value: []byte(v)})
})

// 递归操作, 为了更加理解这个falttenValue方法的作用，举下面这个例子便于理解。
如:
type ClientEvent struct {
    Request    RequestInfo  `trace:"Client.Request"`
    Response   ResponseInfo `trace:"Client.Response"`
    ClientSend time.Time    `trace:"Client.Send"`
    ClientRecv time.Time    `trace:"Client.Recv"`
}

type RequestInfo struct {
    Method        string
    URI           string
    Proto         string
    Headers       map[string]string
    Host          string
    RemoteAddr    string
    ContentLength int64
}

type ResponseInfo struct {
    Headers       map[string]string
    ContentLength int64
    StatusCode    int
}

最终解析结果：key: value = {
        "Client.Request.Headers.Connection":    "close",
        "Client.Request.Headers.Accept":        "application/json",
        "Client.Request.Headers.Authorization": "REDACTED",
        "Client.Request.Proto":                 "HTTP/1.1",
        "Client.Request.RemoteAddr":            "127.0.0.1",
        "Client.Request.Host":                  "example.com",
        "Client.Request.ContentLength":         "0",
        "Client.Request.Method":                "GET",
        "Client.Request.URI":                   "/foo",
        "Client.Response.StatusCode":           "200",
        "Client.Response.ContentLength":        "0",
        "Client.Send":                          "0001-01-01T00:00:00Z",
        "Client.Recv":                          "0001-01-01T00:00:00Z",
}

由上可以看到，如果struct含有trace tag标签，则使用tag：trace作为前缀；否则直接使用字段名作为前缀，且以'.'分割；

// 如果这里的prefix改为prefixKey，更易于理解。
func flattenValue(prefix string, v reflect.Value, f func(k, v string)) {
	// 复杂类型处理，包括time.Time，time.Duration和fmt.Stringer三种类型。
	switch o:=v.Interface().(type) {
	case time.Time:
		f(prefix, o.Format(time.RFC3339Nano))
		return	
	}
	case time.Duration:
		ms := float64(o.Nanoseconds()) / float64(time.Millisecond)
		f(prefix, strconv.FormatFloat(ms, 'f', -1, 64)
		return
	case fmt.Stringer:
		f(prefix, o.String())
		return
	}
	
	// 其他为reflect.Kind的枚举类型
	switch v.Kind() {
	case reflect.Ptr:
		// 指针类型，去指针
		flattenValue(prefix, v.Elem(), f)
	case reflect.Bool:
		// 原子类型
		f(prefix, strconv.FormatBool(v.Bool())
	case reflect.Float32, reflect.Float64:
		// 原子类型
		f(prefix, strconv.FormatFloat(v.Float(), 'f', -1, 64))
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		// 原子类型
		f(prefix, strconv.FormatInt(v.Int(), 10))
	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
		// 原子类型
		f(prefix, strconv.FormatUint(v.Uint(), 10))
	case reflect.String:
		// 原子类型
		f(prefix, v.String())
	case reflect.Struct:
		// 复杂类型
		// 因为做reflect操作比较耗时，所以作者采用的缓存type结构方式
		// var cachedFieldNames map[reflect.Type]map[int]string
		// 我们看到解析过程中，使用了structTag
		// 查找tag为"trace"的标签，并作为prefix追加前缀，否则直接使用字段名作为前缀。
		// nest使用'.'连接和追加前缀
		for i, name := range fieldNames(v) {
			falttenValue(nest(prefix, name), v.Field(i), f)
		}
	case reflect.Map:
		// 复杂类型
		// 这里对于map的处理有点难理解.
		// map结构的key可能为复杂类型，value也可能为复杂类型；
		// 这里是先把key作为复杂类型处理完成后，再处理value类型，先递归key类型，同时保留value类型稍后递归。直到为原子类型或者time.Time、time.Duration或者fmt.Stringer类型。
		for _, key := range v.MapKeys() {
			flattenValue("", key, func(_, k string) {
				flattenValue(nest(prefix, k), v.MapIndex(key), f)
			})
		}
	case reflect.Slice, reflect.Array:
		// 对于列表类型，直接把列表索引号加前缀作为key
		// 比如： Client.Request.XXXX.0
		//       Client.Request.XXXX.1
		//       Client.Request.XXXX....
		// 	      Client.Request.XXXX.(v.Len())
		for i:=0; i < v.Len(); i++{
			flattenValue(nest(prefix, strconv.Itoa(i)), v.Index(i), f)
		}
	default:
		f(prefix, fmt.Sprintf("%+v", v.Interface())
	}
}
```

结合上面的实例和flattenValue序列化，以及理解struct和map的解析，则event序列化为Annotations就理解明白了。

其中：对于map的解析过程有点生涩难懂, 大家可以仔细想想。

## unflattenValue

在正式介绍`unflattenValue`方法之前，需要了解一些辅助方法，例如：`mapToKVs`, `kvsByKey`, `structFieldsByName`, `parseValue` 和 `parseValueToPtr`。

因为Annotations是slice数据类型，而Span携带信息一般都是map结构的，所以存在一个数据转换方法`mapToKVs`：

```shell
func mapToKVs(m map[string]string) *[][2]string {
	var kvs [][2]string // slice{[key, value], ...}
	
	for k, v := range m {
		kvs = append(kvs, [2]string{k, v})
	}
	
	// 把key作为排序字段进行列表排序
	sort.Sort(kvsByKey(kvs))
	return &kvs
}

// 把key作为排序字段进行列表排序
type kvsByKey [][2]string

func (v kvsByKey) Len() int { return len(v) }
func (v kvsByKey) Less(i, j int) { return v[i][0] < v[j][0] }
func (v kvsByKey) Swap(i, j int) { v[i], v[j] = v[j], v[i] }
```

给定一个类型和字符串，转换成指定的数据类型值方法`parseValue`和`parseValueToPtr`：

```shell
func parseValueToPtr(as reflect.Type, s string) (reflect.Value, error) {
	// 复杂类型：time.Time和time.Duration
	switch [2]string{as.PkgPath(), as.Name()} {
	case [2]string{"time", "Time"}:
		// 把时间类型和时间字符串构建一个时间类型值，且时间格式只支持RFC3339Nano。
		t, err := time.Parse(time.RFC3339Nano, s)
		return reflect.ValueOf(&t), nil
	case [2]string{"time", "Duration}:
		// 把time.Duration和字符串值构建成一个时间大小值
		usec, err := strconv.ParseFloat(s, 64)
		d := time.Duration(usec *float64(time.Millisecond))
		return reflect.ValueOf(&d), nil
	}
	
	// reflect.Kind类型
	switch as.Kind() {
	case reflect.Ptr:
		// 去掉类型指针
		return parseValueToPtr(as.Elem(), s)
	case reflect.Bool:
		vv, _ := strconv.ParseBool(s)
		return reflect.ValueOf(&vv), nil
	case reflect.Float32, reflect.Float64:
		vv, _ := strconv.ParseFloat(s, 32)
		switch as.Kind() {
			...// 对于float32和float64的处理
		}
	case reflect.Int, reflect.Int8, ..., reflect.Int64:
		vv, _ := strconv.ParseInt(s, 10, 64)
		switch as.Kind() {
			...// 对于Int，Int8， ..., Int64的处理
		}
	case reflect.Uint, ..., reflect.Uint64:
		... // 同上处理
	case reflect.String:
		return reflect.ValueOf(&s), nil
	}
	return reflect.Value{}, nil
}

// 如果as是指针类型，则直接返回；否则，去指针
func parseValue(as reflect.Type, s string) (reflect.Value, error) {
	vp, err := parseValueToPtr(as, s)
	if as.Kind() == reflect.Ptr {
		return vp, nil
	}
	return vp.Elem(), nil
}
```

接下来，详细看`unflattenValue`方法, 它是反序列化把Annotations的相关数据存放到event类型值中。

```shell
func unflattenValue(prefix string, v reflect.Value, t reflect.Type, kv *[][2]string) error {
	... // 如果kv非有序排列，则直接panic。
	... // 因为map的处理比较特殊，所以直接过滤
	
	if v.IsValid() {
		// 时间类型处理
		treatAsValue := false
		switch v.Interface().(type) {
		case time.Time:
			treatAsValue = true
		}
		
		if treatAsValue {
			// 如果是time.Time类型，则使用上面提供的parseValue方法解析成event数据
			vv, err := parseValue(v.Type(), (*kv)[0][1])
			if vv.Type().AssignableTo(v.Type()) {
				v.Set(vv)
			}
			*kv = (*kv)[1:]
			return nil
		}
	}
	
	// 其他为reflect.Kind类型
	switch t.Kind() {
	case reflect.Ptr:
		// 去指针
		return unflattenValue(prefix, v, t.Elem(), kv)
	case reflect.Interface:
		// 去指针
		return unflattenValue(prefix, v.Elem(), v.Type(), kv)
	case reflect.Bool, reflect.Float32, ... , reflect.Int, ... reflect.String:
		// 解析原子类型
		vv, err := parseValue(v.Type(), (*kv)[0][1])
		v.Set(vv)
		*kv = (*kv)[1:]
	case reflect.Struct:
		// 对于struct中的所有字段都需要排序, 然后再赋值
		if v.Kind() == reflect.Ptr {
			v = v.Elem()
		}
		var vtfs []reflect.StructField
		for i:=0; i < t.NumField(); i++{
			f := t.Field(i)
			vtfs = append(vtfs, f)
		}
		sort.Sort(structFieldsByName(vtfs)) // struct中的所有字段排序,
		// 然后再依次取出kv中的数据处理
		for _, vtf := range vtfs {
			vf := v.FieldByIndex(vtf.Index)
			if vf.IsValid() {
				// 形成前缀, 如果不匹配，则直接跳过kv第一个字段
				fieldPrefix := nest(prefix, fieldName(vtf))
				unflattenValue(fieldPrefix, vf, vtf.Type, kv)
			}
		}
	case reflect.Map:
		m := reflect.MakeMap(t)
		keyPrefix := prefix + "."
		found := 0
		for _, kvv := range *kv {
			key, val := kvv[0], kvv[1]
			// 前缀比较，如果当前形成的keyPrefix比较大，则kv需要跳过
			if key < keyPrefix {
				continue
			}
			// 前缀比较，如果当前形成的keyPrefix小于最小key，同时还不是前缀
			// 则说明接下来的所有key都找不到了， 直接退出
			if key > keyPrefix && !strings.HasPrefix(key, keyPrefix) {
				break
			}
			// 找到相应的key了，并通过map反射写入相应值
			vv, _ :=  parseValue(t.Elem(), val)
			m.SetMapIndex(reflect.ValueOf(strings.TrimPrefix(key, keyPrefix)), vv)
			*kv = (*kv)[1:]
			found++
		}
		// 完成之后再写入到v中
		if found >0 {
			v.Set(m)
		}
	}
	case reflect.Slice, reflect.Array:
		// 思路：先把kv与keyPrefix前缀匹配的数据全部放入到elem列表中
		// 然后，再通过反射把数据写入到v中
		keyPrefix := prefix + "."
		var elems []elem
		maxI := 0
		for _, kvv := range *kv{
			key, val := kvv[0], kvv[1]
			// 如果key不匹配keyPrefix前缀，则说明在kv中无法找到struct，直接退出
			if !strings.HasPrefix(key, keyPrefix) {
				break
			}
			
			// 找到每个struct中资字段的索引，则val就是这个索引对应的字段值
			i, err := strconv.Atoi(strings.TrimPrefix(key, keyPrefix))
			
			elems = append(elems, elem{i, val})
			if i > maxI {
				maxI = i
			}
			
			*kv = (*kv)[1:]
		}
		if v.Kind() == reflect.Slice {
			v.Set(reflect.MakeSlice(t, maxI+1, maxI+1))
		}
		for _, e:= range elems {
			vv, err := parseValue(t.Elem(), e.s)
			v.Index(e.i).Set(vv)
		}
}
```

由上可以看出，`flattenValue`和`unflattenValue`两个方法都是围绕event与annotations的数据转换进行序列化和反序列化的，其中在序列化时，可以看到有些方法已经序列化完成统一的key列表，并存储在内存中，这个是一个很好的优化点。其他的除了map的序列化有点生涩，其他都ok。这里面用到的sort.Sort实现比较多。

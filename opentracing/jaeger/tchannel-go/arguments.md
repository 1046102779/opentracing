# argument

```shell
// ArgReader为OutboundCallResponse和InboundCall读取arg2和arg3数据流
type ArgReader io.ReaderCloser

// ArgWriter是OutboundCall和InboundCallResponse arg2和arg3的数据流
type ArgWriter interface {
	io.WriteCloser
	
	Flush() error
}

// ArgWritable为OuboundCall和InboundCallResponse提供arg2和arg3的写入
//
// 例如：reqResWriter实现了ArgWritable
type ArgWritable interface {
	Arg2Writer() (ArgWriter, error)
	
	Arg3Writer() (ArgWriter, error)
}

// ArgReadable为InboundCall和OutboundCallResponse提供了读取arg2和arg3的interface
//
// 例如：reqResReader实现了ArgReadable
type ArgReadable interface {
	Arg2Reader() (ArgReader, error)
	Arg3Reader() (ArgReader, error)
}

// ArgReadHelper提供了一个读取arguments的简单接口
type ArgReadHeaper struct {
	reader ArgReader
	err error
}

// 这样ArgReader既实现了读取arg2和arg3的接口，又实现了读取错误时的返回
func NewArgReader(reader ArgReader, err error) ArgReadHelper {
	return ArgReadHelper{reader, err}
}

// 这个read方法很有意思
// 
// f函数值为闭包结构，它里面通过读取r.reader数据，并存放到闭包传入的bs *[]byte, 最后再校验r.reader是否已经读取完，否则返回错误。 所以read方法读取arg到外界传入的引用值中，并校验比特流是否读取完成。
//
// 它最大的有点是屏蔽了如何读取数据，并返回给外部的变量值结构也不关心。这就是闭包的最大好处
func (r ArgReaderHelper) read(f func() error) error {
	if r.err != nil {
		return r.err
	}
	
	if err := f(); err != nil {
		return err
	}
	if err := argreader.EnsureEmpty(r.reader, "read arg"); err != nil {
		return err
	}
	
	return r.reader.Close()
}

// 封装读取数据流并校验, 通过闭包函数屏蔽读取比特流和返回值类型的细节
func (r ArgReaderHelper) Read(bs *[]byte) error {
	return r.read(func() error{
		var err error
		*bs, err = ioutil.ReadAll(r.reader)
		return err
	})
}

// 封装读取数据流并解析成json数据
func (r ArgReadHelper) ReadJSON(data interface{}) error {
	return r.read(func() error {
		reader := bufio.NewReader(r.reader)
		// Reader.Peek(n int) 方法是获取前n个字符的引用，pos不会移动。也就是尝试获取一个数据
		// 校验数据流是否已经结束
		if _, err := reader.Peek(1); err == io.EOF {
			return nil
		} else if err !=nil {
			return err
		}
		
		// 数据流解析成json数据流
		d := json.NewDecoder(reader)
		return d.Decode(data)
	})
}

// 与ArgReaderHelper类似，写入argument
// 这里有个设计上的不对称问题，在ArgReaderHelper属性元素有个ArgReader类型，但是这里直接写的io.WriterCloser类型
type ArgWriteHelper struct {
	writer io.WriteCloser
	err error
}

// 获取一个ArgWriter实例
func NewArgWriter(writer io.WriteCloser, err error) ArgWriteHelper {
	return ArgWriteHelper{writer, err}
}

// 和上面的read方法相同，也是通过闭包函数值抽象写入外部变量值和类型。
// 
// 闭包函数值执行完毕后，关闭流
func (w ArgWriteHelper) write(f func() error) error {
	if w.err != nil {
		return w.err
	}
	
	if err := f(); err != nil {
		return err
	}
	
	return w.writer.Close()
}

// 直接比特流写入w
func (w ArgWriteHelper) Write(bs []byte) error {
	return w.write(func() error {
		_, err := w.write.Write(bs)
		return err
	})
}

// data通过json写入到w.writer中
func (w ArgWriteHelper) WriteJSON(data interface{}) error {
	return w.write(func() error {
		e := json.NewEncoder(w.writer)
		return e.Encode(data)
	})
}
```

# 总结

通过Argument相关interface，把argument读取或者写入到指定的流中，它的承载体在Inbound和Outbound中。另外一个大家可以看到的闭包函数值抽象具体实现，并赋值给外部，这个比较有意思。

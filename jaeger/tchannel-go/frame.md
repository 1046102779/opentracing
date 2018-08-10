# Frame

首先需要对tchannel协议规范非常了解，可以看前面翻译的规范。下面是一个简表，通过这个我们在看Frame源码时容易理解

| Position	| Contents| 
|---|---|
|0-7	|size:2 type:1 reserved:1 id:4|
|8-15	|reserved:8|
|16+	|payload - based on type|


我们看到消息帧由header和payload两部分构成，header是固定的16字节，payload为header首2字节大小-16得到结果

固定长度header为16字节，帧最大长度默认无限制

```shell
const (
	MaxFrameSize = math.MaxUint16
	
	FrameHeaderSize = 16
	
	MaxFramePayloadSize = MaxFrameSize - FrameHeaderSize
)
```

## Frame Header

```shell
type FrameHeader struct {
	// 协议帧总长度
	size uint16
	
	// 消息类型共9种, init req, init res, call req ......
	messageType messageType 
	
	// 1字节保留
	reserved1 byte
	
	// 4字节的message id
	ID uint32
	
	8字节的保留
	reserved [8]byte
}

// payload大小加上头部的16字节，为frame的总大小
func (fn *FrameHeader) SetPayloadSize(size uint16) {
	fh.size = size + FrameHeaderSize
}

// 获取payload大小
func (fn *FrameHeader) PayloadSize() uint16 {
	return fn.size - FrameHeaderSize
}

// 返回消息帧的总大小
func (fh FrameHeader) FrameSize() uint16 {
	return fh.size
}

// 提供Format操作, 返回消息类型和消息ID
func (fh *FrameHeader) String() string {
	return fmt.Sprintf("%v[%d]", fh.messageType, fh.ID)
}

// json序列化消息帧
func (fh *FrameHeader) MarshalJSON() ({
	s := struct{
		ID uint32 `json:"id"`
		MsgType messageType `json:"msgType"`
		Size uint16 `json:"size"`
	}{
		fh.Id,
		fh.message.Type,
		fh.size,
	}
	
	return json.Marshal(s)
}

// 我们又碰到了typed.ReadBuffer，在上上节我们详细介绍过, 通过ReadBuffer读取remaining数据
//
// 这里的ReadBuffer中的buffer是完整的协议消息帧
func (fh *FrameHeader) read(r *typed.ReadBuffer) error {
	// 读取header中的首2字节
	fh.size = r.ReadUint16()
	// 读取1字节的消息类型
	fh.messageType = messageType(r.ReadSingleByte())
	// 读取1字节的保留位
	fh.reserve1 = r.ReadSingleByte()
	// 读取4字节的消息ID
	fh.ID = r.ReadUint32()
	// 读取8字节的保留位
	r.ReadBytes(len(fh.reserved))
	
	// 剩余的就是payload部分
	return r.Err()
}

// 与上面一样，碰到了typed.WriteBuffer，通过WriteBuffer写入buffer
func (fh *FrameHeader) write(w *typed.WriteBuffer) error {
	// 写入2字节的帧总大小
	w.WriteUint16(fh.size)
	// 写入1字节的消息类型
	w.WriteSingleByte(byte(fh.messageType))
	// 写入1字节的保留
	w.WriteSingleByte(fh.reserved1)
	// 写入4字节的消息ID
	w.WriteUint32(fh.ID)
	// 写入8字节的保留位
	w.WriteBytes(fh.reserved[:])
	
	return w.Err()
}
```

## Frame 

下面介绍整个消息帧的读写

```shell
// 消息帧结构
type Frame struct {
	// buffer由header和payload两部分构成
	buffer []byte
	// header存储空间
	headerBuffer []byte
	
	// header
	Header FrameHeader
	
	// payload存储空间
	Payload []byte
}

// 创建Frame
func NewFrame(payloadCapacity int) *Frame {
	f := &Frame{}
	// 创建消息帧的内存空间
	f.buffer = make([]byte, FrameHeaderSize + payloadCapacity)
	// payload从16字节开始所属的空间
	f.Payload = f.buffer[FrameHeaderSize:]
	// header为buffer的前16字节空间
	f.headerBuffer = f.buffer[:FrameHeaderSize]
	
	return f
}

// ReadBody方法的第一个参数表示Frame的header，io.Reader表示payload部分
//
// 作用：把header和io.Reader数据写入到Frame的Buffer中
func (f *Frame) ReadBody(header []byte, r io.Reader) error {
	// copy的第一个参数既可以是headerBuffer，也可以是buffer，共享内存
	copy(f.headerBuffer, header)
	
	// 读取header数据到FrameHeader，这样就不用反复解析headerBuffer了
	if err := f.Header.read(typed.NewReadBuffer(header)); err != nil{
		return err
	}
	
	// 读取payload
	switch payloadSize := f.Header.PayloadSize() {
	case payloadSize > MaxFramePayloadSize:
		return fmt.Errorf("invalid frame size %v", f.Header.Size)
	case payloadSize > 0:
		_, err := io.ReadFull(r, f.SizedPayload())
	default:
		return nil
	}
}

// 从io.Reader io流读取一个完整的Frame消息帧
//
// 先读取header，再传入ReadBody解析header和payload到Frame中
// 这里为何不写成一个方法，可能是有很多地方要用到ReadBody?
func (f *Frame) ReadIn(r io.Reader) error {
	header := make([]byte, FrameHeaderSize)
	if _, err := io.ReadFull(r, header); err !=nil{
		return err
	}
	
	return f.ReadBody(header, r)
}

// 这个方法，看源码写得莫名其妙
func (f *Frame) WriteOut(w io.Writer) error {
	var wbuf typed.WriteBuffer
	wbuf.Wrap(f.headerBuffer)
	
	if err := f.Header.Write(&wbuf); err != nil{
		return err
	}
	
	// 以上这段代码写得...
	// 看这份代码好像是要把Header头部写入到Frame的buffer前16字节
	// 但是buffer本身就是已经包含了header和payload。为何还要再写？
	
	fullFrame := f.buffer[:f.Header.FrameSize()]
	if _, err := w.write(fullFrame); err != nil{
		return err
	}
	return nil
}

// 获取payload内存空间
func (f *Frame) SizedPayload() []byte {
	return f.Payload[:f.Header.PayloadSize()]
}

// 返回帧的消息类型
func (f *Frame) messageType() messageType {
	return f.Header.messageType
}

// 把header和payload写入Frame, message interface在前面一节介绍过
func (f *Frame) write(msg message) error {
	// 通过message写入到typed.WriteBuffer
	// 也就写入到了Frame的payload空间
	var wbuf typed.WriteBuffer
	wbuf.Wrap(f.Payload[:])
	if err := msg.write(&wbuf); err != nil{
		return err
	}
	
	// 获取message相关信息，写入Header
	f.Header.ID = msg.ID()
	f.Header.reserve1 = 0
	f.header.messageType = msg.messageType()
	f.Header.SetPayloadSize(uint16(w.buf.BytesWritten()))
}

// 读取Frame的payload内存到msg中
func (f *Frame) read(msg message) error {
	var rbuf typed.ReadBuffer
	rbuf.Wrap(f.SizedPayload())
	return msg.read(&rbuf)
}
```

## 后记

我们可以看到所有的Frame、Header和Payload的数据读取，都是通过typed的WriteBuffer和ReadBuffer底层引用进行读写的。同时Frame也提供了io流和buffer进行读写操作。

通过message interface、WriteBuffer、ReadBuffer、Read和Frame这四节，我们对整个Frame的封装、拆包的讲解全部结束了。

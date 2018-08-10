# ReadBuffer

网络数据流读取, 在上一节，我们知道message的读写操作，全部依赖typed包，这里面就是对数据流的操作。

```shell
// 因为是网络大端模式，所以注意使用encoding/binary包转化大端字节
type ReadBuffer struct {
	// buffer作为内存分配空闲池，如果remaining需要空间，直接从buffer获取
	// remaining作为buffer的内存占用前半部分，剩余未空闲内存
	buffer []byte
	// 当前内存数据流
	remaining []byte
	err error
}

// 使用数据初始化ReadBuffer实例
func NewReadBuffer(buffer []byte) *ReadBuffer {
	return &ReadBuffer{buffer: buffer, remaining: buffer}
}

// 创建一个内存大小为size的ReadBuffer实例
// 
// remaining =nil
func NewReadBufferWithSize(size int) *ReadBuffer {
	return &ReadBuffer{buffer: make([]byte, size), remaining: nil}
}

// 读取1字节的数据
// 
// 与ReadByte方法不同的是，它不关心错误信息，如果读取为空，则数据为0
func (r *ReadBuffer) ReadSingleByte() byte {
	b, _ := r.ReadByte()
	return b
}

// 读取1字节数据, 关心错误
//
// 由这个方法可以看出，数据流内存是remaining，buffer只是保留一份完整的数据。用于恢复或者索引找数据
func (r *ReadBuffer) ReadByte() (byte, error) {
	if r.err != nil {
		return 0, r.err
	}
	
	if len(r.remaining) < 1 {
		r.err = ErrEOF
		return 0, r.err
	}
	
	b := r.remaining[0]
	r.remaining = r.remaining[1:]
	return b, nil
}

// 与ReadByte比较，少了返回参数error。为啥不一致呢？
//
// 读取n个字节数据
func (r *ReadBuffer) ReadBytes(n int) []bytes {
	if r.err != nil {
		return nil
	}
	
	if len(r.remaining) < n {
		r.err = ErrEOF
		return nil
	}
	
	b := r.remaining[:n]
	r.remaining = r.remaining[n:]
	return b
}

// 通过ReadBytes方法获取比特流，然后直接转化为字符串
func (r *ReadBuffer) ReadString(n int) string {
	if b := r.ReadBytes(n); b!=nil {
		// 这里会发生内存拷贝，临时内存
		return string(b)
	}
	
	return ""
}

// 通过ReadBytes获取指定2字节的流后，在通过大端转化uint16数据
func (r *ReadBuffer) ReadUint16() uint16 {
	if b:=r.ReadBytes(2); b!= nil{
		return binary.BigEndian.Uint16(b)
	}
	
	return 0
}

// 通过ReadBytes获取指定4字节的流后，在通过大端转化uint32数据
func (r *ReadBuffer) ReadUint32() uint32 {
	if b := r.ReadBytes(4); b != nil{
		return binary.BigEndian.Uint32(b)
	}
	
	return 0
}

// 通过ReadBytes获取指定8字节的流后，再通过大端转化uint64数据
func (r *ReadBuffer) ReadUint64() uint32 {
	if b := r.ReadBytes(8); b != nil{
		return binary.BigEndian.Uint64(b)
	}
	
	return 0
}

// ReadUvarint reads an unsigned varint from the buffer.
//
// varint编码，不理解  ::TODO
func (r *ReadBuffer) ReadUvarint(r) uint64 {
	v , _ := binary.ReadUvarint(r)
	return v
}

// ReadLen8String方法比较有意思
//
// 它是与tchannel协议帧有关系的，因为在tchannel协议帧中，`nh~2 (key~2 value~2){nh}`类似这样的写法，一般都是块读取，首先获取这块占用内存大小，然后再整体读取
func (r *ReadBuffer) ReadLen8String() string {
	// 比如：头部用1字节存储整个块的大小, 先读取1字节，获取长度
	n := r.ReadSingleByte()
	// 在通过得到的字节长度，读取n个字节，并以字符串形式返回
	return r.ReadString(int(n))
}

func (r *ReadBuffer) ReadLen16String() string {
	// 头部用2字节存储块大小
	n := r.ReadUint16()
	// 读取n字节数据
	return r.ReadString(int(n))
}

// 获取内存剩余字节数
func (r *ReadBuffer) BytesRemaining() int {
	return len(r.remaining)
}

// 从io.Reader读取n字节数据流，并存储到remaining
func (r *ReadBuffer) FillFrom(ior io.Reader, n int) (int, error ){
	if len(r.buffer) < n {
		return 0, r.ErrEOF
	}
	
	// 从这里，我们可以知道
	// len(remaining)：buffer[:len(remaining)] 已填充内存数据
	// buffer[len(remaining):] 空闲内存池
	r.err = nil
	r.remaining = buffer[:n]
	
	return io.ReadFull(ior, r.remaining)
}

func (r *ReadBuffer) Err() error {
	return r.err
}
```

# WriteBuffer

WriteBuffer是把内存数据写入到网络数据流中，与ReadBuffer相反。也是在上节message里用到，作为协议帧的消息类型流写入

```shell
// 结构和ReadBuffer相同
type WriteBuffer struct {
	// buffer前半段是内存数据占用地方， 空闲内存索引index，正时remaining索引为0的地方。
	buffer []byte
	remaining []byte
	err error
}

// 通过buffer初始化WriteBuffer实例
func NewWriteBuffer(buffer []byte) *WriteBuffer {
	return &WriteBuffer{
		buffer: buffer,
		remaining: buffer,
	}
}


// 通过传入初始化的内存大小，获取WriteBuffer实例
func NewWriteBufferWithSize(size int) *WriteBuffer {
	return NewWriteBuffer(make([]byte, size))
}

// 写入1字节到buffer中。
func (w *WriteBuffer) WriteSingleByte(n byte) {
	if w.err != nil {
		return
	}
	
	if len(w.remaining) < 1 {
		w.err = ErrBufferFull
		return
	}
	
	w.remaining[0] = n
	// 从这里，我们可以看出:
	// buffer前半段已经填充内存数据流，remaining索引为0，这是buffer空闲内存开始的地方。
	w.remaining = w.remaining[1:]
	return
}

// 从remaining获取n字节大小的空闲内存
func (w *WriteBuffer) reserve(size int) []byte {
	if w.err != nil{
		return nil
	}
	
	if len(w.remaining) < size {
		w.err = ErrBufferFull
		return nil
	}
	
	b := w.remaining[:n]
   w.remaining = w.remaining[n:] 
   return b
}

// 写入len(in)字节内存
func (w *WriteBuffer) WriteBytes(in []byte) {
	if b := w.reserve(len(in)); b != nil{
		// 内存数据拷贝到b中
		copy(b, in)
	}
	return
}

// 写入2字节数据
func (w *WriteBuffer) WriteUint16(n uint16) {
	if b := w.reserve(2); b != nil{
		binary.BigEndian.PutUint16(b,n)
	}
}

// 写入4字节数据
func (w *WriteBuffer) WriteUint32(n uint32) {
	if b := w.reserve(4); b != nil{
		binary.BigEndian.PutUint32(b, n)
	}
}

// 写入8字节数据
func (w *WriteBuffer) WriteUint64(n uint64) {
	if b := w.reserve(8); b != nil{
		binary.BigEndian.PutUint64(b, n)
	}
}

// varint编码，不理解  ::TODO
func (w *WriteBuffer) WriteUvarint(n uint64) {
	... 
}

// 这个写入字符串，长度通过len获取，存在问题，因为中文、英文占用长度不一样
func (w *WriteBuffer) WriteString(s string) {
	// 这里不能直接使用WriteBytes方法，会存在两次拷贝
	if b := w.reserve(len(s)); b!=nil{
		copy(b, s)
	}
}

// 首先写入字符串占用大小，然后再写入块数据
func (w *WriteBuffer) WriteLen8String(s string) {
	// 写入1字节大小长度
	w.WriteSingleByte(byte(len(s))
	// 写入块
	w.WriteString(s)
}

func (w *WriteBuffer) WriteLen16String(s string) {
	w.WriteBytes(uint16(len(s)))
	w.WriteString(s)
}

// 获取比特数据空闲的引用
func (w *WriteBuffer) DeferByte() ByteRef {
	if len(w.remaining) == 0 {
		w.err = ErrBufferFull
		return ByteRef(nil)
	}
	
	w.remaining[0] = 0
	// 这里大家可能会有误解，
	// 这里好像是把remaining全部空间都给ByteRef， 其实即使ByteRef滥用除index=0的空间外，是不生效的， 因为remaining还是从index=0开始写数据，认为其他都是空闲区
	// 所以写为w.remaining[0:1]或者w.remaining[0:]都是一样的
	bufRef := ByteRef(w.remaining[0:]) 	
	w.remaining = w.remaining[1:]
	return bufRef
}

// 获取n字节大小的空闲内存，并初始化
func (w *WriteBuffer) deferred(size int) []byte {
	bs := w.reserve(n)
	for i := range bs {
		bs[i] = 0
	}
	return bs
}

// 获取2字节空间，并初始化
func (w *WriteBuffer) DeferUint16() Uint16Ref {
	return Uint16Ref(w.deferred(2))
}

// 获取4字节空间，并初始化
func (w *WriteBuffer) DeferUint32() Uint32Ref {
	return Uint32Ref(w.deferred(4))
}

// 获取n字节空间，并初始化
func (w *WriteBuffer) DeferBytes(n int) BytesRef {
	return BytesRef(w.deferred(n))
}

// 获取剩余空闲内存空间
func (w *WriteBuffer) BytesRemaining() int {
	return len(w.remaining)
}

// 拷贝buffer已写入数据的内存空间到io.Writer
func (w *WriteBuffer) FlushTo(iow io.Writer) (int, error) {
	dirty := w.buffer[:w.BytesWritten()]
	return iow.Write(dirty)
}

// 返回已经写入数据的占用内存空间大小，是在buffer前面部分
func (w *WriteBuffer) BytesWritten() int {
	return len(w.buffer) - len(w.remaining)
}

// 把整个buffer内存空间赋值给remaining，则remaining又可以从buffer索引为0的位置开始写入
//
// 不要管脏数据，通过长度和索引可以控制读写
func (w *WriteBuffer) Reset() {
	w.remaining = w.buffer
	w.err = nil
}

func (w *WriteBuffer) Err() error {
	return w.err
}

// 初始化内存空间, 指向传入的空间
func (w *WriteBuffer) Wrap(b []byte) {
	w.buffer = b
	w.remaining = b
}
```

# buffer reference

对于写入协议帧的payload部分，需要频繁申请内存空间，WriteBuffer首先划分了一块内存，然后payload中的某个字段写入需要内存，则从划分好的内存块中获取，获取到的内存是已经初始化过的，全部比特位为0.

## ByteRef

```shell
// 1字节空间
type ByteRef []byte

// 更新内存值
func (ref ByteRef) Update(b byte) {
	if ref != nil {
		ref[0] = b
	}
}
```

## Uint16Ref

```shell
// 2字节空间
type Uint16Ref []byte

// 更新内存值
func (ref Uint16Ref) Update(n uint16) {
	if ref != nil{
		binary.BigEndian.PutUint16(n)
	}
}
```

## Uint32Ref

```shell
// 4字节空间
type Uint32Ref []byte

// 更新内存值
func (ref Uint32Ref) Update(n uint32) {
	if ref != nil {
		binary.BigEndian.PutUint32(n)
	}
}
```

## Uint64Ref

```shell
// 8字节空间
type Uint64Ref []byte

// 更新内存值
func (ref Uint64Ref) Update(n uint64) {
	if ref != nil {
		binary.BigEndian.PutUint64(n)
	}
}
```

## BytesRef

```shell
// 多字节空间
type BytesRef []bytes

// 更新内存值
func (ref BytesRef) Update(b []byte) {
	if ref != nil{
		copy(ref, b)
	}
}

// 使用字符串更新内存值
func (ref BytesRef) UpdateString(s string) {
	if ref != nil [
		// 这个copy是builtin内建包函数
		// 
		// 这个copy原型特别的支持字符串拷贝
		// As a special case, it also will copy bytes from a string to a slice of bytes.
		copy(ref, s)
	}
}
```

# 小结

我们可以看到8, 16, 32和64位是大端模式，但是字符串则没有考虑大端。

从这节的ReadBuffer和WriteBuffer，我们可以看到对于tchannel协议帧payload部分的各个字段读写，都是由一块内存Buffer分配对象和回收对象的。这样可以防止小内存空间频繁操作导致内存消耗过大。

这个Buffer部分，大家可以借鉴，很有意义。里面还提到了string(buf)的临时拷贝是应该避免的。

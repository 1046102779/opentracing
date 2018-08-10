# Reader

该Reader提供一个32字节的内存空间，用于读写小字节的数据；每次读写都是从index=0的位置开始。一次性读取，反复使用;如果超过32字节的空间，则直接使用外界存储。

所以这个Reader一般用于小字节空间分配，超过maxPoolLenStringLen大小，则该Reader无意义。

```shell
const maxPoolLenStringLen = 32

// 把io.Reader的数据读取指定字节到buf中，每次读取io.Reader指定字节并写入到buf中。
type Reader struct {
	reader io.Reader
	err error
	buf [maxPoolStringLen]byte
}

// 临时对象池
var readerPool = sync.Pool {
	New: func() interface{}{
		return &Reader{}
	}
}

// 从临时对象池readerPool获取一个Reader对象
func NewReader(reader io.Reader) *Reader {
	r :=readerPool.Get().(*Reader)
	
	r.reader = reader
	r.err = nil
}

// 从io.Reader中读取2字节数据
func (r *Reader) ReadUint16() uint16 {
	if r.err != nil{
		return 0
	}
	
	// 从buf中获取2字节空间
	buf := buf[:2]
	
	var readN int
	readN, r.err = io.ReadFull(r.reader, buf)
	// io.Reader不够2字节
	if readN < 2 {
		return 0
	}
	
	// 大端读写2字节数据
	return binary.BigEndian.Uint16(buf)
}

// 从io.Reader中读取指定大小的字符串
func (r *Reader) ReadString(n int) string {
	if r.err != nil{
		return ""
	}
	
	var buf []byte
	if n <= maxPoolStringLen {
		buf = buf[:n]
	} else {
		buf = make([]byte, n)
	}
	
	var readN int
	readN, r.err = io.ReadFull(r.reader, buf)
	// io.Reader的数据长度不够n
	if readN < n {
		return ""
	}
	
	// 拷贝一次
	return string(buf)
}

// 和payload部分读取数据类似。先读取头部获取占用内存大小，然后再读取实际数据
func (r *Reader) ReadLen16String() string {
	// 读取2个字节，获取真实数据长度
	n := r.ReadUint16()
	return r.ReadString(n)
}

func (r *Reader) Err() error {
	return r.err
}

// 释放临时对象Reader
func (r *Reader) Release() {
	readerPool.Put(r)
}
```

# 小结

与上节ReaderBuffer不同的是, 上节是数据内存块空间，而这小节是使用的io.Reader文件流形式。同时，该Reader适合32字节的内存临时空间，不适合超过该字节空间。

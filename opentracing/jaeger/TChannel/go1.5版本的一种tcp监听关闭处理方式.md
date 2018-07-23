tchannel-go项目作者prashantv对golang1.5 linux版本socket Accept方法的封装


## 问题描述

作者在linux上测试tchannel-go项目时发现，当关闭服务端的listener后，有时候还是有一些client connection能够进来。后来作者对当时的go1.5版本进行大量测试，确实发现存在，当服务端主动关闭后，client还能够进来。但是在osx其他平台不会发现这个问题。

当listener做关闭操作后，然后client再发起请求建立连接。实际上它只是标记socket为Closed状态，但是不会影响epoll接收新连接。

```shell
详细解释：

如果epoll所在的监听队列上有新来的连接，这时socket accept正在从该队列上获取新来的连接。这时，如果server主动关闭listener，则因为server端存在连接引用，所以暂时不会关闭，需要等待accept当前新来的连接处理完成后，再关闭并destory fd。

所以出现该问题的主要原因是，当accept正在获取新来的连接时，因为引用计数不为0，所以导致监听无法真正关闭。只有当前accept获取到新来的连接后，才会使得引用计数降为0，则时才会真正关闭监听。
```

## 问题解决

所以针对这个go1.5版本linux等存在的缺陷，需要在外层引入引用计数和条件变量，当accept正在阻塞或获取新来的连接时，如果server直接关闭监听，则正在阻塞状态的goroutine，直接收到server关闭error；如果正在获取新来的连接，则外层加一个引用计数，当获取完成后，在减去这个引用计数；在server的close方法包装一层，如果这个引用计数不为0，则阻塞当前这个goroutine，直到引用计数等于0，再退出。这样保证了accept不会再接收到新的连接。

### 代码示例

```shell
// 对net.Listener的封装，引入引用计数和条件变量
type SaneListener struct {
	l net.Listener
	c *sync.Cond
	refCount int
}

// 当进入Accept之前，引用计数做加一操作，防止server主动Close listener操作时，因为底层的引用计数不为0，导致暂时不会发生真正的close fd操作。
// SaleListener的Close操作，因为refCount引用计数不为0，则Close暂时不会退出。底层的监听已关闭，但是会等待accept获取新连接操作处理完成，这样close操作就相当于滞后了一个连接处理。
func (s *SaneListener) incRef() {
	s.c.L.Lock()
	s.refCount++
	s.c.L.Unlock()
}

// 当accept获取到新来的连接或者获取到一个server监听关闭error，引用计数减一
// 这样SaneListener的Close操作，因为引用计数关闭，则真正关闭。
// 
// 由于SaneListener的Close操作，如果server正在获取新连接，则该goroutine会发生条件阻塞；等待accept操作完成后，通过Broadcast操作唤醒睡眠的goroutine，继续Close操作。
func (s *SaneListener) decRef() {
	s.c.L.Lock()
	s.refCount---
	s.c.Broadcast()
	s.c.L.Unlock()
}

// accept操作
func (s *SaneListener) Accept() (net.Conn, error){
	s.incRef()
	defer s.decRef()
	return s.l.Accept()
}

// Close操作：底层的监听是提前关闭了，但是epoll队列中正在被accept的新连接还尚在处理中，所以底层的引用计数不等于0，则需要该操作完成后，再退出Close调用。这样，在Close操作后，server不会接收新的连接了
func (s *SaneListener) Close() error {
	err := s.l.Close()
	if err == nil {
		s.c.L.Lock()
		for s.refCount > 0 {
			s.c.Wait()
		}
		s.c.L.Unlock()
	}
	return err
}

func (s *SaneListener) Addr() net.Addr {
	return s.l.Addr()
}
```

条件变量cond，使得refCount大于0时，主动阻塞该goroutine；等待accept完成获取新连接或者获取到error操作后，在通过decRef的Broadcast广播唤醒阻塞的goutines。

## 小结

我们可以看到server对监听关闭操作，当accept正在获取新来的连接时，因引用计数不为0，则不会真正的destroy掉net fd。通过引入上层引用计数，来达到当关闭监听后，确保server不会再接收新连接了。这个引用计数和条件变量大家可以认真学习，同时学习下golang的网络库。


## 参考资料

[tchannel-go net.listener](https://github.com/uber/tchannel-go/blob/dev/tnet/listener.go)

[net: Listener sometimes accepts connections after Close](https://github.com/golang/go/issues/13762)

[Golang网络:核心API实现剖析(一)](https://juejin.im/entry/5a24c5456fb9a044fa19b086)

[关于TCP 半连接队列和全连接队列](http://jm.taobao.org/2017/05/25/525-1/)

## 后记

```shell
// go1.10.2/src/internal/poll/fd_unix.go 第93行
// 我们可以看到当底层accept获取新连接的引用计数为0时，才会真正destory掉net fd。

 77 // Close closes the FD. The underlying file descriptor is closed by the
 78 // destroy method when there are no remaining references.
 79 func (fd *FD) Close() error {
 80     if !fd.fdmu.increfAndClose() {
 81         return errClosing(fd.isFile)
 82     }
 83
 84     // Unblock any I/O.  Once it all unblocks and returns,
 85     // so that it cannot be referring to fd.sysfd anymore,
 86     // the final decref will close fd.sysfd. This should happen
 87     // fairly quickly, since all the I/O is non-blocking, and any
 88     // attempts to block in the pollDesc will return errClosing(fd.isFile).
 89     fd.pd.evict()
 90
 91     // The call to decref will call destroy if there are no other
 92     // references.
 93     err := fd.decref()
 94
 95     // Wait until the descriptor is closed. If this was the only
 96     // reference, it is already closed. Only wait if the file has
 97     // not been set to blocking mode, as otherwise any current I/O
 98     // may be blocking, and that would block the Close.
 99     if !fd.isBlocking {
100         runtime_Semacquire(&fd.csema)
101     }
102
103     return err
104 }

// go1.10.2/src/internal/poll/fd_mutex.go
209 func (fd *FD) decref() error {
210     if fd.fdmu.decref() {
211         return fd.destroy()
212     }
213     return nil
214 }
```

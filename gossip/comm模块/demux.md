demux
===

demux文件中定义了channel的多路选择器。

# Channel结构

```golang
type channel struct {
	pred common.MessageAcceptor
	ch chan interface{}
}
```

# ChannelDeMultiplexer类

ChannelDeMultiplexer类接受channel注册和分发，并将分发根据预设条件广播给相应的注册

## 属性

ChannelDeMultiplexer包含channel数组、锁等属性

```golang
type ChannelDeMultiplexer struct {
	channels []*channel
	lock *sync.RWMutex
	closed int32
}
```

## 构造方法

### NewChannelDemultiplexer方法

创建一个新的ChannelDeMultiplex对象

## 普通方法

### isClosed方法

返回m.closed是否为1

### Close方法

关闭ChannelDeMultiplexer的所有通道

### AddChannel方法

注册一个包含预设条件的channel

参数：

- predicate common.MessageAcceptor

返回值：

- chan interface{}

```golang
m.lock.Lock()
defer m.lock.Unlock()
ch := &channel{ch: make(chan interface{}, 10), pred: predicate}
m.channels = append(m.channels, ch)
return ch.ch
```

### DeMultiplex方法

广播消息到AddChannel返回的并且持有预设条件的所有通道


```golang
defer func() {
	recover()
}()

if m.isClosed() {
	return 
}

m.lock.RLock()
channels := m.channels
m.lock.RUnlock()

for _, ch := range channels {
	if ch.pred(msg) {
		ch.ch <- msg
	}
}
```
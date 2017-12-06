payloads_buffer
===

定义了PayloadsBuffer接口及实现

# PayloadsBuffer接口

PayloadsBuffer用于存储payload，用于支持根据序号重排区块的payload，同时支持在期望区块到达时发送信号


```golang
type PayloadsBuffer interface {
	Push(payload *proto.Payload) error
	Next() uint64
	Pop() int
	Size() int
	Ready() chan struct{}	
	Close()
}
```

- Push方法：将新区块添加到buffer中
- Next方法：返回下一个期望的序号
- Pop方法：移除并返回给定需要的payload
- Size方法：返回当前buffer大小
- Ready方法：返回传输期望序号的payload到达的信号
- Close方法：关闭

# PayloadsBufferImpl类

## 属性

包含存储payload的map，readyChan等属性

```golang
type PayloadsBufferImpl struct {
	next uint64
	buf map[uint64]*proto.Payload
	readyChan chan struct{}
	mutex sync.RWMutex
	logger *logging.Logger
}
```

## 构造函数

### NewPayloadsBuffer方法

创建新的payloads buffer的工厂方法

```golang
return &PayloadsBufferImpl {
	buf: make(map[uint64]*proto.Payload),
	readyChan: make(chan struct{}, 0),
	next: next,
	logger: util.GetLogger(util.LoggingStateModule, "")
}
```

## 普通方法

### Ready方法

返回readyChan。表明期望的区块到达，一个安全pop out下一个区块

### Push方法

将新的payload添加到buffer中。如果payload的序号低于期望的高度，丢弃之，并返回错误。

将payload添加到b.buf，并新建线程，向b.readyChan发送信号

### Next方法

返回b.next

### Pop方法

从b.buf中取得b.next的消息并移除，next加1

### Size方法

返回len(b.buf)

### Close方法

关闭b.readyChan
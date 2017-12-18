batcher
===

batcher定义了batchingEmitter接口及实现，用于gossip push和forwarding阶段

# 全局变量与函数定义

## emitBatchCallback函数类型

发送batch的回调函数类型

```golang
type emitBatchCallback func([]interface)
```

# batchedMessage结构

包含一个对象了从左开始的迭代器

```golang
type batchedMessage struct {
	data 			interface{}
	interationsLeft int
}
```

# batchingEmitter接口

batchingEmitter接口用于gossip pull/forwarding阶段。添加消息到batchingEmitter，然后以T为时间周期按批进行转发；如果batchingEmitter存储消息的大小到达某个值，也会触发消息分发

```golang
Add(interface{})
Stop()
Size() int
```

- Add方法：添加打包的消息
- Stop方法：停止此模块
- Size方法：返回还未发送的消息的条目

# batchingEmitterImpl结构

batchingEmitterImpl是batchingEmitter接口的实现。包含emitBatchCallback、[]*batchedMessage等属性

## 属性

```golang
type batchingEmitterImpl struct {
	iterations 	int
	burstSize 	int
	delay 		time.Duration
	cb 			emitBatchCallback
	lock 		*sync.Mutex
	buff 		[]*batchedMessage
	stopFlag 	int32
}
```

## 构造方法

### newBatchingEmitter方法

创建一个新的batchingEmitter对象

参数：

- iterations int：每个消息转发的次数
- burstSize int：由消息条目大小触发转发的阀值
- latency time.Duration：每个消息在转发前被存储的最长时间
- cb emitBatchCallback：发送消息的回调函数

返回值：

- batchingEmitter

首先创建一个batchingEmitter对象。如果iterations不等于0，新建go routine，调用p.periodicEmit()定期发送。

## 普通方法

### periodicEmit方法

如果!p.toDie(),一直执行p.emit()发送

### emit方法

将p.buff中的数据存储在msgs2beEmitted中，调用p.cb(msgs2beEmitted)处理回调函数，调用p.decrementCounters()减少计数

### decrementCounters方法

减少计数,并移除iterationsLeft为0的消息

```golang
n := len(p.buff)
for i:=0;i<n;i++{
	msg := p.buff[i]
	msg.iterationsLeft--
	if msg.iterationsLeft == 0 {
		p.buff = append(p.buff[:i],p.buff[i+1:]...)
		n--
		i--
	}
}
```

### toDie方法

返回p.stopFlag是否为1

### Stop方法

将p.stopFlag设置为1

### Size方法

返回p.buff的长度

### Add方法

将消息添加到p.buff中
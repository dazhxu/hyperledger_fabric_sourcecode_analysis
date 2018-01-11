solo
===

solo包定义了solo模式的共识器

# consenter结构

consenter结构定义了共识器。其属性为空。

## 构造方法

为solo共识模式创建一个新的共识器。solo共识模式非常简单，对于一个给定的chain只允许一个共识器。它接受通过Enqueue发送来的消息，排序，然后使用blockcutter形成区块，最后写入给定的账本。返回一个空的consenter对象(multichain.Consenter类型)。

## HandleChain方法

调用newChain方法返回一个multichain.Chain

参数：

- support multichain.ConsenterSupport
- metadata *cb.Metadata

# chain结构

chain结构定义了solo的链

## 属性

chain包含ConsenterSupport、cb.Envelope、exitChan等属性

```golang
type chain struct {
	support multichain.ConsenterSupport
	sendChan chan *cb.Envelop
	exitChan chan struct{}
}
```

## 构造方法

### newChain方法

创建一个chain对象

参数：

- support multichain.ConsenterSupport

## 普通方法

### Start方法

启动一个go routine，调用ch.main方法

### main方法

定义一个timer定时器。

处理信号。

如果是ch.sendChan中的消息msg，获取batches和committers。如果batcher为0，并且timer为空，创建一个时长为ch.support.SharedConfig().BatchTimeout()的定时器。否则将batch写块。

如果是timer的信号，如果batch不为空，创建一个区块。

如果是ch.exitChan的信号，退出。

### Halt方法

关闭ch.exitChan

### Enqueue方法

接受一个消息，如果接受返回true，否则返回false

```golang
select {
case ch.sendChan <- env:
	return true
case <- ch.exitChan:
	return false
}
```

### Errored方法

返回ch.exitChan
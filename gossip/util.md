util模块
===

util模块为其他上层模块提供辅助功能，由logging、misc、msgs和pubsub四个文件组成。

# logging

logging文件定义了gossip模块中其他子模块的日志名字，用一个map来存储每个模块的Logger对象。

```golang
var loggersByModules = make(map[string]*logging.Logger)
```

另外，实现了GetLogger方法，别的模块可以调用此方法获取相应模块的日志对象。

# misc

misc文件定义了一个Set对象，包含一个线程安全的set容器，并包含Add、Exists、Size、Clear、Remove等基本方法。

```golang
type Set struct {
	item map[interface{}]struct{}
	lock *sync.RWMutex
}
```

另外，还定义了一些随机方法，如GetRandomIndices返回一个从0到hignestIndex区间固定个数的随机不重复的索引；RandomInt返回一个[0,n)区块的随机数。

其他函数，GetIntOrDefault和GetDurationOrDefault从viper对象中获取一个int或时间段。

# msgs

msgs文件定义了MembershipStore对象，封装了membership消息存储的抽象类型，存储了从pkiID到消息的映射。

```golang
type MembershipStore struct {
	m map[string]*proto.SignedGossipMessage
	sync.RWMutex
}
```

其中SignedGossipMessage在fabric/proto/gossip/extensions.go文件中定义，封装了GossipMessage及包含GossipMessage的Envelope

```golang
type SignedGossipMessage struct {
	*Envelope
	*GossipMessage
}
```

MembershipStore对象包含以下方法：

- MsgByID: 返回特定ID的消息；如果不存在，返回nil
- Size：返回memship store的大小
- Put：将消息与给定的pkiID关联起来，存储在memship store中
- Remove：移除给定pkiID的消息。
- ToSlice: 返回存储的所有消息。

# pubsub

pubsub对象定义了Subscription接口及实现，PubSub对象。

## Subscription接口及实现

Subscription接口定义了可以接受publish的topic的订阅，包括Listen方法。Subscription接口的实现如下：

```golang
type subscription struct {
	top string
	ttl time.Duration
	c chan interface{}
}
```

**Listen方法**一直阻塞到一个publish到了，或者在TTL耗尽时返回一个错误。

## PubSub对象

PubSub对象可以分发item到有多个subscription的topic、从topic上订阅消息。

```golang
type PubSub struct {
	sync.RWMutex
	subscriptions map[string]*Set
}
```

- Publish方法

Publish方法分发一个item到topic上的所有subscription。

获取此topic的subscription，如果subscription的管道缓存未满，则将item发送到subscription的管道中。

- Subcribe方法

新建一个subscription对象，并订阅到给定的topic中。将unSubscribe方法作为ttl过期后的处理函数。

```golang
time.AfterFunc(ttl, func{
	ps.unSubcribe(sub)
})
```

- unSubscribe方法

将给定的subscription从topic中移除。

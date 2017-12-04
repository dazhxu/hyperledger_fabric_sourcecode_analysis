discovery_impl
===

discovery_impl实现了discovery接口

# 常量、变量及其操作

## 常量

```golang
const defaultHelloInterval = time.Duration(5) * time.Second
const msgExpirationFactor = 20
```

defaultHelloInterval指探测远端节点的心跳频率
msgExpirationFactor指消息失效的因子

## 变量

```golang
var aliveExpirationCheckInterval time.Duration
var maxConnectionAttemps = 120
```

## 方法

### SetAliveExpirationTimeout方法

设定expiration的超时时间

### SetAliveExpirationCheckInterval方法

设定失效检查的间隔

### SetReconnectInterval方法

设定重连的间隔

### SetMaxConnAttempts方法

设置最大的重试次数

### getAliveTimeInterval

获取保活消息的时间间隔

### getAliveExpirationTimeout方法

获取保活失效的超时时间

### getAliveExpirationCheckInterval方法

获取保活失效的检查时间间隔

### getReconnectInterval方法

获取重连的时间间隔

# timestamp类

## 属性

```golang
type timestamp struct {
	incTime time.Time
	seqNum uint64
	lastSeen time.Time
}
```

## 相关方法

### before方法

判断两个时间戳那个靠前

```golang
func before(a *timestamp, b *proto.PeerTime) bool
```

# aliveMsgStore类

## 属性

```golang
type aliveMsgStore struct {
	msgstore.MessageStore
}
```

其中msgstore是在gossip/gossip/msgstore模块中定义的

## 构造方法

### newAliveMsgStore方法

新建一个保活信息的MsgStore

```golang
func newAliveMsgStore(d *gossipDiscoveryImpl) *aliveMsgStore {
	policy := proto.NewGossipMessageComparator(0)
	trigger := func(m interface{}) {}
	aliveMsgTTL := getAliveExpirationTimeout() * msgExpirationFactor
	externalLock := func() { d.lock.Lock() }
	externalUnlock := func() { d.lock.Unlock() }
	callback := func(m interface{}) {
		msg := m.(*proto.SignedGossipMessage)
		if !msg.IsAliveMsg() {
			return
		}
		id := msg.GetAliveMsg().Membership.PkiId
		d.aloveMembership.remove(id)
		d.deadMembership.Remove(id)
		delete(d.id2Member, string(id))
		delete(d.deadLastTS, string(id))
		delete(d.aliveLastTS, string(id))
	}
	s := &aliveMsgStore {
		MessageStore: msgstore.NewMessageStoreExpirable(policy, trigger, aliveMsgTTL, externaLock, externalUnlock, callback),
	}
	return s
}
```

## 普通方法

### Add方法

添加消息到aliveMsgStore

### CheckValid方法

调用MessageSotre的CheckValid方法校验消息

# gossipDiscoveryImpl类

## 属性

gossipDiscoveryImpl的属性包括incTime、seqNum、id2Member、aliveMembership、deadMembership、comm等属性

- incTime uint64: 当前时间的unix时间
- seqNum uint64: 序号
- self NetworkMember: 自身的网络成员信息
- deadLastTS map[string]*timestamp: 
- aliveLastTS map[string]*timestamp:
- id2Member make(map[string]*NetworkMember): 存储从PKIid到NetworkMember的映射关系
- aliveMembership *util.MembershipStore: 存储alive成员信息
- deadMembership *util.MembershipStore: 存储dead成员信息
- msgStore *aliveMsgStore: 存储alive消息
- comm CommService: 通信服务接口
- crypt CryptoService: 加密服务接口
- lock *sync.RWMutex: 读写锁
- toDieChan chan struct{}: 传送失联节点的通道
- toDieFlag int32
- port: 端口
- logger *logging.Logger
- disclosurePolicy DisclosurePolicy：关闭策略
- pubsub *util.PubSub:分发订阅对象

## 构造方法

### NewDiscoveryService方法

新建discoveryImpl对象

参数：

- self NetworkMember: 实例自身的网络成员信息
- comm CommService：通信服务接口
- crypt CryptoService: 加密服务接口
- disPol DisclosurePolicy: 关闭策略

首先创建一个gossipDiscoveryImpl对象

```golang
d := &gossipDiscoveryImpl {
	self: self,
	incTime: uint64(time.Now().UnixNano()),
	seqNum: uint64(0),
	deadLastTS: make(map[string]*timestamp),
	aliveLastTS: make(map[string]*timestamp),
	id2Member: make(map[string]*NetworkMember),
	aliveMembership:  util.NewMembershipStore(),
	deadMembership:   util.NewMembershipStore(),
	crypt:            crypt,
	comm:             comm,
	lock:             &sync.RWMutex{},
	toDieChan:        make(chan struct{}, 1),
	toDieFlag:        int32(0),
	logger:           util.GetLogger(util.LoggingDiscoveryModule, self.InternalEndpoint),
	disclosurePolicy: disPol,
	pubsub:           util.NewPubSub(),
}
```

然后调用d.validateSelfConfig检查自身配置；并调用newAliveMsgStore新建msgStore存储保活信息

然后，启动新线程周期性发送保活信息；启动新线程周期性检查节点活性；启动新线程处理消息；启动新线程周期性连接到失联节点；启动信息线程，处理失联的节点

```golang
go d.periodicalSendAlive()
go d.periodicalCheckAlive()
go d.handleMessages()
go d.periodicalReconnectToDead()
go d.handlePresumedDeadPeers()
```

## 普通方法

### validateSelfConfig方法

校验自身的配置



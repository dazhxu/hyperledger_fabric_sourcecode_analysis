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

校验自身的配置。主要是校验endpoint的格式

### periodicalSendAlive方法

周期性的发送保活消息。首先调用d.createAliveMessage(true)新建一个alive消息，然后调用d.comm.Gossip(msg)将消息gossip出去

### createAliveMessage方法

新建alive消息。

参数：

- includeInternalEndpoint bool: 是否包含内部endpoint

返回值：

- *proto.SignedGossipMessage
- error

首先，将d.seqNum加一；然后创建一个GossipMessage.

```golang
msg2Gossip := &proto.GossipMessage{
	Tag: proto.GossipMessage_EMPTY,
	Content: &proto.GossipMessage_AliveMessage{
		AliveMsg: &proto.AliveMessage{
			Membership: &proto.Member{
				Endpoint: endpoint,
				Metadata: metadata,
				PkiId: pkiID,
			},
			TimestampL &proto.PeerTime{
				IncNum:uint64(d.incTime),
				SeqNum: seqNum,
			},
		}
	}
}
```

然后调用CryptoService的SignMessage方法对消息进行签名，创建SignedGossipMessage

```golang
envp := d.crypt.SignMessage(msg2Gossip, internalEndpoint)
...
SignedMsg := &proto.SignedGossipMessage{
	GossipMessage: msg2Gossip,
	Envelope: envp,
}
if !includeInternalEndpoint {
	signedMsg.Envelope.SecretEnvelop = nil
}
return signedMsg, nil
```

### periodicalCheckAlive方法

周期检查存活节点。

首先调用d.getDeadMembers方法获取失联的存活节点，然后调用expireDeadMembers方法检查失联节点活性

### getDeadMembers方法

获取失联节点的pkiID

返回值：

- []common.PKIidType: 失联节点的pkiID

对于aliveLastTS中的每个条目，获取从最近可见的时间，计算失联的时间长度。如果失联的实际长度大于getAliveExpirationTime获得的时间，则认为节点失联，将其PKIid加入dead数组中。返回。

### expireDeadMember方法

使失联节点失效

参数：

- dead []common.PKIidType

将deadMember从aliveLastTS移到deadLastTS,从aliveMembership存储移到deadMembership存储。

调用d.comm.CloseConn(member2Expire)关闭到deadMember的链接

### handlerMessages方法

处理消息接受到的消息。首先调用d.comm.Accept获取传送消息的通道，然后针对不同的信号进行处理。如果是d.toDieChan的信号，将信号传送到d.toDieChan；如果是消息通道的信号，调用d.handleMsgFromCommc处理消息

```golang
in := d.comm.Accept()
for !d.toDie() {
	select {
	case s:= <-d.toDieChan:
		d.toDieChan <- s
		return
	case m := <- in:
		d.handleMsgFromComm(m)
	}
}
```

### handleMsgFromComm方法

处理消息。消息可能是Alive消息，或者是MembershipResponse消息，或者是MembershipRequest消息

参数：

- m *proto.SignedGossipMessage：收到的消息

(1) 如果是MembershipRequest消息，将自身的信息创建一个Gossip消息，然后检查消息的有效性，如果有效调用d.handleAliveMessage处理消息。最后，新启线程，调用d.sendMemresponse发送消息

```golang
if memReq := m.GetMemReq(); memReq != nil {
	selfInfoGossipMsg, err := memReq.SelfInformation.ToGossipMessage()
	...
	if d.msgStore.CheckValid(selfInfoGossipMsg) {
		d.handleAliveMessage(selfInfoGossipMsg)
	}

	var internalEndpoint string
	if m.Envelope.SecretEnvelope != nil {
		internalEndpoint = m.Envelope.SecretEnvelope.InternalEndpoint()
	}

	go d.sendMemResponse(selfInfoGossipMsg.GetAliveMsg().Membership, internalEndpoint, m.Nonce)
	return
}
```

(2) 如果是AliveMsg，将消息添加到msgStore，调用handleAliveMessage处理消息，然后调用d.comm.Gossip广播消息。

```golang
if m.IsAliveMsg() {
	if !d.msgStore.Add(m) {
		return
	}
	d.handleAliveMessage(m)
	d.comm.Gossip(m)
	return
}
```

(3) 如果是MemshipResponse消息，首先，将消息的Nonce分发出去；然后对于Alive消息，检查消息的有效性，调用d.handleAliveMessage处理消息；对于Dead消息，调用d.learnNewMembers处理

```golang
if memResp := m.GetMemRes(); memResp != nil {
	d.pubsub.Publish(fmt.Sprintf("%d", m.nonce), m.Nonce)

	for _,env := range memResp.Alive {
		am, err := env.ToGossipMessage()
		...
		if d.msgStore.CheckValid(am) {
			d.handleAliveMessage(am)
		}
	}

	for _, env := range memResp.Dead {
		dm, err := env.ToGossipMessage()
		...
		if !d.crypt.ValidateAliveMsg(dm) {
			continue
		}
		if !d.msgStore.CheckValid(dm) {
			return
		}
		newDeadMembers := []*proto.SignedGossipMessage{}
		d.lock.RLock()
		if _, known := d.id2Member[string(dm.GetAliveMsg().Membership.PkiId)]; !know {
			newDeadMembers = append(newDeadMembers, dm)
		}
		d.lock.RUnlock()
		d.learnNewMembers([]*proto.SignedGossipMessage{}, newDeadMembers)
	}
}
```

### handleAliveMessage方法

处理Alive消息

参数：

- m *proto.SignedGossipMessage

首先调用d.crypt.ValidateAliveMsg方法校验消息。

如果收到消息的pkiID与解答自身的pkiID相同，比较消息和自身的InternalEndpoint或者ExternalEndpoint是否相同。

```golang
pkiID := m.GetAliveMsg().Membership.PkiId
if equalPKIid(pkiID, d.self.PKIid) {
	diffExternalEndpoint := d.self.Endpoint != m.GetAliveMsg().Membership.Endpoint
	var diffExternalEndpoint bool
	secretEnvelope := m.GetSecreteEnvelope()
	if secretEnvelope != nil && secreteEnvelope.InteralEndpoint() != "" {
		diffExternalEndpoint = secreateEnvelope.InternalEndpoint() != d.self.InternalEndpoint
	}
	if diffInternalEndpoint || diffExternalEndpoint {
		d.logger.Error("...")
	}
	return
}
```

如果不是自身，首先获取m.GetAliveMsg().Timestamp。查找id2Member,如果没找到，调用d.learnNewMembers方法处理。然后查找d.aliveLastTS和deadLastTS。如果在deadLastTS中，如果lastDeadTS在ts之前，调用d.resurrectMember恢复节点。如果在aliveLastTS中，如果lastAliveTS在ts之前，调用learnExistMembers进行处理

### resurrectMember方法

恢复dead节点

参数：

- am *proto.SignedGossipMessage
- t proto.PeerTime

首先，从消息重获取成员信息，新建pkiID到timestamp的映射

```golang
member := am.GetAliveMsg().Membership
pkiID := memberPkiId
d.aliveLastTS[string(pkiID)] = &timestamp {
	lastSeen: time.Now(),
	seqNum: t.SeqNum,
	incTime: tsToTime(t.IncNum)
}
```
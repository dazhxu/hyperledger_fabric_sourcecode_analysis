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

更新远端节点成员信息，从d.deadLastTS中删除节点的pki信息，并从deadMembership移到aliveMembership

### learnNewMembers方法

感知新的节点信息

参数：

- aliveMembers []*proto.SignedGossipMessage
- deadMembers []*proto.SignedGossipMessage

对于aliveMembers，如果am.GetAliveMsg().Membership.PkiId不是节点自身的，在aliveLastTS创建条目，并将其放入aliveMembership中；
对于deadMembers，如果am.GetAliveMsg().Membership.PkiId不是节点自身的，在deadLastTS创建条目，并将其放入deadMembership中；

无论在哪种情况下，都更新id2Member的信息

### learnExistingMembers方法

感知已存在的成员

参数：

- aliveArr []*proto.SignedGossipMessage

对于aliveArr的每个元素，首先更新id2Member中的信息。如果消息中的成员在deadLastTS或不在aliveLastTS中，说明节点已经过期，进行下一个元素。否则，更新aliveLastTS存在的活性数据，如果aliveMembership中不存在，添加到其中；否则替换信息。

### sendMemResponse方法

发送成员回应信息

参数：

- targetMember *proto.Member
- internalEndpoint string
- nonce uint64

首先，创建目标peer结构，然后调用createAliveMessage方法创建alive消息，接着调用createMembershipResponse方法创建回应；将回应打包成proto.SingedGossipMessage，然后调用d.comm.SendToPeer将消息发送出去

```golang
targetPeer := &NetworkMember{
	Endpoint:			targetMember.Endpoint,
	Metadata:			targetMember.Metadata,
	PKIid:				targetMember.PkiId,
	InternalEndpoint:	internalEndpoint,
}
aliveMsg, err := d.createAliveMessage(true)
...
memResp := d.createMembershipResponse(aliveMsg, targetPeer)
if memResp == nil {
	d.comm.CloseConn(targetPeer)
	return
}
...
msg, err := (&proto.GossipMessage{
	Tag: 	proto.GossipMessage_EMPTY,
	Nonce: 	nonce,
	Content:&proto.GossipMessage_MemRes{
		MemRes: memResp,
	},
}).NoopSign()
...
d.comm.SendToPeer(targetPeer, msg)
```

### createAliveMessage方法

创建Alive消息

参数：

- includeInternalEndpoint bool

返回值：

- *proto.SignedGossipMessage
- error

根据d.self的信息创建GossipMessage，然后调用d.crypt.SignMessage对消息进行签名，并打包成proto.SignedGossipMessage消息

### createMembershipResponse

创建MembershipResponse

参数：

- aliveMsg *proto.SignedGossipMessage
- targetMember *NetworkMember

返回值：

- *proto.MembershipResponse

调用d.disclosurePolicy判断是否应该被关闭。获取deadPeer和alivePeer

```golang
shouldBeDisclosed, omitConcealedFields := d.disclosurePolicy(targetMember)

if !shouldBeDisclosed(aliveMsg) {
	return nil
}
...
deadPeers := []*proto.Envelope{}
for _, dm := range d.deadMembership.ToSlice() {
	if !shouldBeDisclosed(dm) {
		continue
	}
	deadPeers = append(deadPeers, omitConcealedFields(dm))
}

var aliveSnapshot []*proto.Envolope
for _, am := range d.aliveMembership.ToSlice() {
	if !shouldBeDisclosed(am) {
		continue
	}
	aliveSnapshot = append(aliveSnapshot, omitConcealedFields(am))
}

return &proto.MembershipResponse{
	Alive: append(aliveSnapshot, omitConcealedFields(aliveMsg)),
	Dead: deadPeers,
}
```

### periodicalReconnectToDead方法

周期性地连接dead节点。

对于deadLastTS中的每个节点，新启一个线程，检查可达性。如果Ping通，调用d.sendMembershipRequest方法，否则返回。

```golang
for !d.toDie() {
	wg := &sync.WaitGroup{}

	for _, member := range d.copyLastSeen(d.deadLastTS) {
		wg.Add(1)
		go func(member NetworkMember) {
			defer wg.Done()
			if d.comm.Ping(&member) {
				d.sendMembershipRequest(&member, true)
			} else {
				d.logger.Debug(member, "is still dead")
			}
		} (member)
	}

	wg.Wait()
	time.Sleep(getReconnectInterval())
}
```

### sendMembershipRequest方法

发送成员信息请求消息。

参数：

- member *NetworkMember
- includeInternalEndpoint bool

调用d.createMembershipRequest方法创建消息，然后调用d.comm.SendToPeer将请求发送

```golang
m, err := d.createMembershipRequest(includeInternalEndpoint)
...
req, err := m.NoopSign()
...
d.comm.SendToPeer(member, req)
```

### createMembershipRequest方法

创建Membership请求

首先，调用createAliveMessage消息。然后创建proto.MembershipRequest方法，其中Known字段设置为空字节，以防远端节点不该知道其他节点信息。返回proto.GossipMessage消息

```golang
am, err := d.createAliveMessage(includeInternalEndpoint)
...
req := &proto.MembershipRequest{
	SelfInfomation: am.Envelope,
	Known: [][]byte{},
}
return &proto.GossipMessage{
	Tag: proto.GossipMessage_EMPTY,
	Nonce: uint64(0),
	Content: &proto.GossipMessage_MemReq{
		MemReq: req,
	},
}, nil
```

### copyLastSeen方法

从id2Member中收集lastSeen节点的Member信息

```golang
res := []NetworkMember{}
for pkiIDStr := range lastSeenMap {
	res = append(res, *(d.id2Member[pkiIDStr]))
}
return res
```

### handlePresumedDeadPeers方法

处理被认为是dead的节点。

对于d.comm.PresumedDead()通道的信号，调用d.isAlive方法，如果判断为真，调用d.expireDeadMembers方法失效节点；
对于d.toDieChan的信号，发送信号到d.toDieChan

```golang
for !d.toDie() {
	select {
	case deadPeer := <-d.comm.PresumedDead():
		if d.isAlive(deadPeer) {
			d.expireDeadMembers([]common.PKIidType{deadPeer})
		}
	case s := <-d.toDieChan:
		d.toDieChan <- s
		return
 	}
}
```

### isAlive方法

从d.aliveLastTS中查询节点是否存活。

### Lookup方法

从id2Member中查询PKIid对应的NetworkMember

### Connect方法

不停的重试连接远端节点。

参数：

- member NetworkMember
- id identifier

如果member是自身，直接返回。

新启线程，进行尝试重连。每次重连首先调用d.createMembershipRequest方法创建成员信息请求，并调用util.RandomUint64为请求消息创建一个Nonce，最后新启线程调用d.sendUntilAcked方法发送请求

### sendUntilAcked方法

尝试发送消息，直到确认

参数：

- peer *NetworkMember:
- message *proto.SignedGossipMessage

创建一个topic为nonce的订阅，将message发送到peer，如果没有超时，直接返回；否则等待一段时间，重新发送

```golang
nonce := message.Nonce
for i := 0; i < maxConnectionAttempts && !d.toDie(); i++ {
	sub := d.pubsub.Subscribe(fmt.Sprintf("%d",nonce), time.Seconde*5)
	if _, timeoutErr := sub.Listen(); timeoutErr == nil {
		return
	}
	time.Sleep(getReconnectInterval())
}
```

### InitiateSync方法

初始化同步，在aliveMembership中随机选择min(peerNum, d.aliveMembership.Size())个节点，发送membershiprequest

### GetMembership方法

从aliveMembership中获取成员信息

返回值：

- []NetworkMember

```golang
response := []NetworkMember{}
for _, m := range d.aliveMembership.ToSlice() {
	member := m.GetAliveMsg()
	response = append(response, NetworkMember{
		PKIid: 				member.Membership.PkiId,
		Endpoint: 			member.Membership.Endpoint,
		Metadata: 			member.Membership.Metadata,
		InternalEndpoint: 	d.id2Member[string(m.GetAliveMsg).Membership.PkiId].InternalEndpoint,
	})
}
return response
```

### UpdateMetadata方法

更新metadata

### UpdataEndpoint方法

更新endpoint

### Self方法

返回自身的NetworkMember

### Stop方法

停止discoveryImpl对象。

首先设置d.toDieFlag为1，然后调用d.msgStore.Stop()停止msgStore，最后想d.toDieChan中发送一个空对象
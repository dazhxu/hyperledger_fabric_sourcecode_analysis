channel
===

channel文件定义了gossip通道相关的结构与操作

# Config结构

Config结构定义了channel store的configuration item

```golang
type Config struct {
	ID 							string
	PublishStateInfoInterval 	time.Duration
	MaxBlockCountToStore 		int
	PullPeerNum 				int
	PullIntreval 				time.Duration
	RequestStateInfoInterval 	time.Duration
	BlockExpirationInterval 	time.Duration
	StateInfoCachSweepInterval 	time.Duration
}
```

# Adapter接口

Adapter接口使得gossipChannel与gossipServiceImpl进行通信

```golang
type Adapter interface {
	GetConf() Config
	Gossip(message *proto.SignedGossipMessage)
	DeMultiplex(interface{})
	GetMembership() []discovery.NetworkMember
	Lookup(PKIID common.PKIidType) *discovery.NetworkMember
	Send(msg *proto.SingedGossipMessage, peers ...*comm.RemotePeer)
	ValidateStateInfoMessage(message *proto.SignedGossipMessage) error
	GetOrgOfPeer(pkiID common.PKIidType) api.OrgIdentityType
	GetIdentityByPKIID(pkiID common.PKIidType) api.PeerIdentityType
}
```

- GetConf方法：返回GossipChannel拥有的configuration
- Gossip方法：在通道中gossip消息
- DeMultiplex方法：分用一个item到订阅
- GetMembership方法：返回已知的存活节点及它们的信息
- Lookup方法：返回一个network member
- Send方法： 发送一个消息到一组peer
- ValidateStateInfoMessage方法：如果消息没有被正确签名返回一个error，否则返回nil
- GetOrgOfPeer方法：返回给定peer pkiID的组织id
- GetIdentityByPKIID方法：返回拥有特定pkiID的节点的identity，如果没有找到返回nil

# stateInfoCache类

stateInfoCache实际上是一个索引消息的messageStore，从而使消息一个在后面被提取出来

## 属性

```golang
type stateInfoCache struct {
	*util.MembershipStore
	msgstore.MessageStore
	stopChan chan struct{}
}
```

## 构造方法

### newStateInfoCache方法

创建一个stateInfoCache对象

参数：

- sweepInterval time.Duration：擦除时间间隔
- hasExpired func(interface{}) bool: 判断是否过期

返回值：

- *stateInfoCache：stateInfoCache对象

首先，调用util.NewMembershipStore方法创建一个membershipStore。

创建消息替换策略，pol := proto.NewGossipMessageComparator(0)；

然后创建invalidationTriggger

```golang
invalidationTrigger := func(m interface{}) {
	pkiID := m.(*proto.SignedGossipMessage).GetStateInfo().PkiId
	membershipStore.Remove(pkiID)
}
```

根据pol和invalidationTrigger创建s.MessageStore = msgstore.NewMessageStore(pol, invalidationTrigger)

创建新的go routine处理信号。如果是s.stopChan的信号，返回；如果是time.After(sweepInterval)信号，执行s.Purge(hasExpired)失效消息

## 普通方法

### Add方法

将给定的消息添加到stateInfoCache中，如果添加成功，加你索引。

```golang
added := cache.MessageStore.Add(msg)
if added {
	pkiID := msg.GetStateInfo().PkiId
	cache.MembershipStore.Put(pkiID, msg)
}
```

### Stop方法

发送信号到stopChan

# membershipFilter结构

## 属性

```golang
type membershipFilter struct {
	adapter Adapter
	*gossipChannel
}
```

## GetMembership方法

返回已知存活节点及他们的信息

```golang
for _, mem := range mf.adapter.GetMembership() {
	if mf.eligibelForChannelAndSameOrg(mem) {
		members = append(members, mem)
	}
}
```

# GossipChannel接口

GossipChannel是处理所有与channel相关的消息的对象

```golang
type GossipChannel interface {
	GetPeers() []discovery.NetworkMember
	IsMemberInChan(member discovery.NetworkMember) bool
	UpdateStateInfo(msg *proto.SignedGossipMessage)
	IsOrgInChannel(membersOrg api.OrgIndentityType) bool
	EligibleForChannel(member discovery.NetworkMember) bool
	HandleMessage(proto.ReceivedMessage)
	AddToMsgStore(msg *proto.SignedGossipMessage)
	ConfigureChannel(joinMsg api.JoinChannelMessage)
	Stop()
}
```

- GetPeers方法：返回分发到通道的带有metadata的一组peer
- IsMemberInChan方法：检查给定的member是否是在通道里
- UpdateStateInfo方法：更新通道的StateInfo消息，这些消息是周期性进行分发
- IsOrgInChannel方法：给定的组织是否在Channel中
- EligibleForChannel方法：返回给定的member是否应该从此channel获取区块
- HandleMessage方法：处理远端节点发送的消息
- AddToMsgStore方法：将给定的Gossip消息添加到message store
- ConfigureChannel方法：配置channel中合法组织的列表
- Stop方法：停止channel活动

# gossipChannel结构

## 属性

gossipChannel是对GossipChannel接口的实现，继承Adapter接口，包含api.MessageCryptoService, pkiID, selfOrg, stateInfoMsg, joinMsg, blockMsgStore, stateInfoMsStore, leaderMsgStore, chainID, blocksPuller等属性

```golang
type gossipChannel struct {
	Adapter
	sync.RWMutex
	shouldGossipStateInfo 		int32
	mcs 						api.MessageCryptoService
	pkiID 						common.PKIidType
	selfOrg 					api.OrgIdentityType
	stopChan                  chan struct{}
	stateInfoMsg              *proto.SignedGossipMessage
	orgs                      []api.OrgIdentityType
	joinMsg                   api.JoinChannelMessage
	blockMsgStore             msgstore.MessageStore
	stateInfoMsgStore         *stateInfoCache
	leaderMsgStore            msgstore.MessageStore
	chainID                   common.ChainID
	blocksPuller              pull.Mediator
	logger                    *logging.Logger
	stateInfoPublishScheduler *time.Ticker
	stateInfoRequestScheduler *time.Ticker
	memFilter                 *membershipFilter
}
```

## 构造函数

### NewGossipChannel方法

创建一个新的GossipChannel

参数：

- pkiID common.PKIidType：节点的pkiID
- org api.OrgIdentityType: 机构id
- mcs api.MessageCryptoService：消息加密服务接口
- chainID commom.ChainID: 通道名字
- adapter Adapter：Adapter接口
- joinMsg api.JoinChannelMessage：

返回值：

- GossipChannel方法

首先根据参数创建一个gossipChannel对象gc。设置gc.memFilter=&membershipFileter{adapter:gc.Adapter, gossipChannel: gc}。

创建blockMsgStore

```golang
comparator := proto.NewGossipMessageComparator(adapter.GetConf().MaxBlockCountStore)
gc.blocksPuller = gc.createBlockPuller()
seqNumFromMsg := func(m interface{}) string {
	return fmt.Sprintf("%d",m.(*proto.SignedGossipMessage).GetDataMsg().Payload.SeqNum)
}
gc.blockMsgStore = msgstore.NewMessageStoreExpirable(comparator, func(m interface{}) {
	gc.blocksPuller.Remove(seqNumFromMsg(m))
}, gc.GetConf().BlockExpirationInterval, nil, nil, func(m interface{}) {
	gc.blocksPuller.Remove(seqNumFromMsg(m))
})
```

设置gc.stateInfoMsgStore。

```golang
hasPeerExpiredInMembership := func(o interface{}) bool {
	pkiID := o.(*proto.SignedGossipMessage).GetStateInfo().PkiId
	return gc.Lookup(pkiID) == nil
}
gc.stateInfoMsgStore = newStateInfoCache(gc.GetConf().StateInfoCacheSweepInterval, hasPeerExpiredInMembership)
```

设置gc.leaderMsgStore

```golang
ttl := election.GetMsgExpirationTimeout()
pol := proto.NewGossipMessageComparator(0)
gc.leaderMsgStore = msgStore.NewMessageStoreExpirable(pol, msgstore.Noop. ttl, nil, nil, nil)
```

调用gc.ConfigureChannel(joinMsg)配置通道

新启线程，调用gc.periodicalInvocation(gc.publishStateInfo, gc.statInfoPublishScheduler.C)周期性地分发stateinfo
新启线程，调用调用gc.periodicalInvocation(gc.requestStateInfo, gc.statInfoRequestScheduler.C)周期性地请求stateinfo

## 普通方法

### createBlockPuller方法

创建区块puller

返回值：

- pull.Mediator

首先创建pull.Config

```golang
conf := pull.Config{
	MsgType: proto.PullMsgType_BLOCK_MSG,
	Channel: []byte(gc.chainID),
	ID: gc.GetConf().ID,
	PeerCountToSelect: gc.GetConf().PullPeerNum,
	PullInterval: gc.GetConf().PullInterval,
	Tag: proto.GossipMessage_CHAN_AND_ORG,
}
```

创建IdExtractor函数，从消息中获取序号。并根据idExtractor创建PullAdapter

```golang
seqNumFromMsg := func(msg *proto.SignedGossipMessage) string {
	dataMsg := msg.GetDataMsg()
	if dataMsg == nil || dataMsg.Payload == nil {
		return ""
	}
	return fmt.Sprintf("%d",dataMsg.Payload.SeqNum)
}
adapter := &pull.PullAdapter{
	Sndr: gc,
	MemSvc: gc.memFilter,
	IdExtractor: seqNumFromMsg,
	MsgCons: func(msg *proto.SignedGossipMessage) {
		gc.DeMultiplex(msg)
	},
}
```

最后，调用pull.NewPullMediator(conf, adapter)创建BlockPuller

### ConfigureChannel方法

ConfigChannel在channel中配置合法的组织的列表

参数：

- joinMsg api.JoinChannelMessage

如果gc.joinMsg.SequenceNumber()>(join.SequenceNumber())说明通道拥有更新的JoinChannel消息，直接返回；否则设置gc.orgs=joinMsg.Members(),gc.joinMsg=joinMsg。

### periodicalInvocation方法

定期调用方法。

参数：

- fn func(): 定期调用的函数
- c <-chan time.Time：周期定时器通道

如果是c中的信号，调用fn()方法；如果是gc.stopChan中的信号，将空对象发送到gc.stopChan，并返回

### Stop方法

停止channel操作。

首先向gc.stopChan发送一个空对象。停止gc.blocksPuller，停止gc.stateInfoPulishScheduler，停止gc.stateInfoRequestScheduler，停止gc.leaderMsgInfoMsgStore，停止gcstateInfoMsgStore，停止gc.blockMsgStore.

### GetPeers方法

返回分发到channel中的peer的列表

返回值：

- []discovery.NetworkMember

通过gc.GetMembership()获取成员，对于每个成员，如果!gc.EligibleForChannel(member)，continue；调用stateInf.GetStateInfo().Metadata获取member的stateInf，然后获取Metadata, member.Metadata=stateInf.GetStateInfo().Metadata，将member加入members并返回。

### requestStateInfo方法




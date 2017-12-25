gossip_impl
===

gossip_impl定义了gossipServiceImpl结构及相关的结构，是gossip层的实现

# 常量、全局变量与类型定义

## 常量

```golang
const (
	presumedDeadChanSize = 100 //被认为是断开的chan的大小
	accpectChanSize      = 100 //可接收的chan的大小
)
```

## 变量

```golang
identityExpirationCheckInterval = time.Hour * 24 //每24小时检查失效的节点
identityInactivityCheckInterval = time.Minute * 10 //每10分钟检查不活跃的身份
```

## channelRoutingFilterFactory函数类型

用来创建filter.RoutingFilter的工厂方法

```golang
type channelRoutingFilterFactory func(channel.GossipChannel) filter.RoutingFilter
```

# discoveryAdapter结构

dicoveryAdapter为discovery模块提供其需要的通信接口

## 属性

discoveryAdapter包含传输dead节点pkiid的通道、传输SignedGossipMessage的通道等属性

```golang
type discoveryAdapter struct {
	stopping 			int32
	c 					comm.Comm
	presumedDead 		chan common.PKIidType
	incChan 			chan *proto.SignedGossipMessage
	gossipFunc   		func(message *proto.SignedGossipMessage)
	disclosurePolicy 	discovery.DisclosurePolicy
}
```

## 构造方法

### newDiscoveryAdapter方法

newDiscoveryAdapter方法属于gossipServiceImpl结构的方法，创建一个新的discoveryAdapter实例。

```golang
func (g *gossipServiceImpl) newDiscoveryAdapter() *discoveryAdapter {
	return &discoveryAdapter {
		c: 	g.comm,
		stopping: int32(0),
		gossipFunc: func(msg *proto.SignedGossipMessage) {
			if g.conf.PropagateIterations == 0 {
				return
			}
			g.emitter.Add(msg)
		},
		incChan: make(chan *proto.SignedGossipMessage),
		presumedDead: g.presumedDead,
		disclosurePolicy: g.disclosurePolicy,
	}
}
```

## 普通方法

### close方法

将da.stopping设置为1，并close(da.incChan)

### toDie方法

返回da.stopping是否为1

### SendToPeer方法

发送消息到某个peer

参数：

- peer *discovery.NetworkMember
- msg *proto.SignedGossipMessage

首先检查知道PKIid的节点的成员请求。不知道他们pkiid的节点只有bootstrap节点

```golang
if memReq :=msg.GetMemReq(); memReq != nil && len(peer.PKIid) != 0 {
	selfMsg, err := memReq.SelfInfomation.ToGossipMessage()
	...
	_, omitConcealedFields := da.disclosurePolicy(peer)
	selfMsg.Envelope = omitConcealedFields(selfMsg)
	oldKnown := memReq.Known
	memReq = &proto.MembershipRequest{
		SelfInfomation: selfMsg.Envelope,
		Known: oldKnown,
	}
	msg.Content = &proto.GossipMessage_MemReq{
		MemReq: memReq,
	}
	msg, err = msg.NoopSign()
	...
}
```

调用comm层的Send接口发送消息

```golang
da.c.Send(msg, &comm.RemotePeer{PKIID: peer.PKIid, Endpoint: peer.PreferredEndpoint()})
```

### Ping方法

调用comm层的Probe方法探测节点活性

### Accept方法

返回传输SignedGossipMessage的专用通道

```golang
return da。incChan
```

### PresumedDead方法

返回传输dead节点pkiid的通道

```golang
return da.presumedDead
```

### CloseConn方法

调用comm层的CloseConn方法关闭到某个特定peer的连接

# discoverySecurityAdapter结构

discoverySecurityAdapter的为discovery模块提供安全的通信

## 属性

discoverySecurityAdapter拥有identity、idMapper、comm.Comm等属性

```golang
type discoverySecurityAdapter struct {
	identity 				api.PeerIdentityType
	includeIdentityPeriod 	time.Time
	idMapper 				identity.Mapper
	sa 						api.SecurityAdvisor
	mcs 					api.MessageCryptoService
	c 						comm.Comm
	logger 					*logging.Logger
}
```

## 构造方法

### newDiscoverySecurityAdapter方法

newDiscoverySecurityAdapter方法属于gossipServiceImpl结构，新建一个discoverySecurityAdapter对象

## 普通方法

### ValidateAliveMsg方法

校验一个alive消息是否合法

参数：

- m *proto.SignedGossipMessage

返回值：

- bool

如果消息中包含身份，将pkiid和identity放入idMapper；否则从idMapper中获取identity。调用sa.validateAliveMsgSignature(m, identity)校验签名。

### SignMessage方法

签名一个AliveMessage并更新它的signature域

参数：

- m *proto.GossipMessage
- internalEndpoint string

返回值：

- * proto.Envelope

首先初始化话signer函数

```golang
signer := func(msg []byte) ([]byte, error) {
	return sa.mcs.Sign(msg)
}
```

如果消息IsAliveMsg且时间在sa.includeIdentityPeriod之前，将m.GetAliveMsg().Identity设置为sa.identity。

创建proto.SignedGossipMessage，并用signer进行签名

```golang
sMsg := &proto.SignedGossipMessage{
	GossipMessage: m,
}
e, err := sMsg.Sign(signer)
```

最后，根据SignedGossipMessage签名secret envelope

```golang
e.SignSecret(signer, &proto.Secret{
	Content: &proto.Secret_InternalEndpoint{
		InternalEndpoint: internalEndpoint,
	},
})
```

### validateAliveMsgSignature方法

校验alive消息的签名。

参数：

- m *proto.SignedGossipMessage
- identity api.PeerIdentityType

返回值：

- bool

首先调用m.GetAliveMsg()获取alive消息。然后创建校验器。

```golang
verifier := func(peerIdentity []byte, signature, message []byte) error {
	return sa.mcs.Verify(api.PeerIdentityType(peerIdentity),signature, message)
}
```

校验消息，返回结果

```golang
err := m.Verify(identity, verifier)
if err != nil {
	return false
}
return true
```

# gossipServiceImpl结构

gossipServiceImpl结构是对GossipService的实现

## 属性

gossipServiceImpl结构包含如下属性

```golang
type gossipServiceImpl struct {
	selfIdentity 			api.PeerIdentityType  //节点自身身份
	includeIdentityPeriod 	time.Time   //身份有效时间
	certStore 				*certStore  //存储身份
	idMapper 				identity.Mapper //存储pkiid到identity的映射
	presumedDead 			chan common.PKIidType //dead节点的通道
	disc 					discovery.Discovery //discovery实例
	comm 					comm.Comm //comm实例
	incTime 				time.Time //
	selfOrg 				api.OrgIdentityType //组织身份
	*comm.ChannelDeMultiplexer //通道分用实例
	logger 					*logging.Logger
	stopSignal 				*sync.WaitGroup
	conf 					*Config
	toDieChan 				chan struct{}
	stopFlag 				int32
	emitter 				batchingEmitter  //区块/消息发送器
	discAdapter 			*discoveryAdapter
	secAdvisor 				api.SecurityAdvisor
	chanState 				*channelState
	disSecAdap 				*discoverySecurityAdapter
	mcs 					api.MessageCryptoService
	stateInfoMsgStore 		msgstore.MessageStore
}
```

## 构造方法

### NewGossipService方法

创建一个gossip实例，附加到一个gRPC服务器上

参数：

- conf *Config: 配置类实例
- s *gprc.Server: grpc服务器
- secAdvisor api.SecurityAdvisor: 
- mcs api.MessageCryptoService
- idMapper identity.Mapper:
- selfIdentity api.PeerIdentityType
- secureDialOpts api.PeerSecureDialOpts

返回值：

- Gossip

首先创建Comm层

```golang
if s == nil {
	c, err = createCommWithServer(conf.BindPort, idMapper, selfIdentity, secureDialOpts)
} else {
	c, err = createCommWithoutServer(s, conf.TLSServerCert, idMapper, selfIdentity, secureDialOpts)
}
```

然后创建一个gossipServiceImpl对象

```golang
g := &gossipServiceImpl{
	selfOrg:               secAdvisor.OrgByPeerIdentity(selfIdentity),
	secAdvisor:            secAdvisor,
	selfIdentity:          selfIdentity,
	presumedDead:          make(chan common.PKIidType, presumedDeadChanSize),
	idMapper:              idMapper,
	disc:                  nil,
	mcs:                   mcs,
	comm:                  c,
	conf:                  conf,
	ChannelDeMultiplexer:  comm.NewChannelDemultiplexer(),
	logger:                lgr,
	toDieChan:             make(chan struct{}, 1),
	stopFlag:              int32(0),
	stopSignal:            &sync.WaitGroup{},
	includeIdentityPeriod: time.Now().Add(conf.PublishCertPeriod),
}
```

然后，创建gossipServiceImpl的其他属性

```golang
g.stateInfoMsgStore = g.newStateInfoMsgStore()
g.chanState = newChannelState(g)
g.emitter = newBatchingEmitter(...)
g.discAdapter = g.newDiscoveryAdapter()
g.disSecAdap = g.newDiscoverySecurityAdapter()
g.disc = discovery.NewDiscoveryService(g.selfNetworkMember, g.discAdapter, g.disSecAdap, g.disclosurePolicy)
g.certStore = newCertStore(g.createCertStorePuller(), idMapper, )
```

最后，新启goroutine，调用g.start()开始gossip服务；新启goroutine，调用g.periodocalValidationAndExpiration()定期进行验证和失效工作；新启go routine，调用g.connect2BootstrpPeers()连接到bootstrap节点

## 普通方法

### createCommWithServer方法

创建一个comm层的server

参数：

- port int
- idStore identity.Mapper
- identity api.PeerIdentityType
- secureDialOpts api.PeerSecureDialOpts

返回值：

- comm.Comm
- error

调用comm模块的NewCommInstanceWithServer转接Comm实例

### createCommWithoutServer方法

调用comm层的NewCommInstance方法创建一个comm层的server

### newStateInfoMsgStore方法

调用msgstore.NewMessageStoreExpirable方法，创建一个MessageStore实例

### newChannelState方法

此方法是gossip_impl模块的全局方法，目的是创建一个channelState

### createCertStorePuller方法

创建新的pullMediator实例。

首先创建pull.Config

```golang
conf := pull.Config{
	MsgType:           proto.PullMsgType_IDENTITY_MSG,
	Channel:           []byte(""),
	ID:                g.conf.InternalEndpoint,
	PeerCountToSelect: g.conf.PullPeerNum,
	PullInterval:      g.conf.PullInterval,
	Tag:               proto.GossipMessage_EMPTY,
}
```

创建从消息中获取pkiID的函数pkiIDFromMsg；创建certConsumer方法，从消息中获取identity，放入g.idMapper。

创建PullAdapter

```golang
adapter := &pull.PullAdapter{
	Sndr: g.comm,
	MemSvc: g.disc,
	IdExtractor: pkiIDFromMsg,
	MsgCons: certConsumer,
	DigFilter: g.sameOrgOrOurPullFilter,
}
```

创建pullMediator并返回

```golang
return pull.NewPullMediator(conf, adapter)
```

### sameOrgOrOurOrgPullFilter方法

如果peer是本组织的，gossip所有的身份

```golang
peersOrg := g.secAdvisor.OrgByPeerIdentity(msg.GetConnectionInfo().Identity)
...
if bytes.Equal(g.selfOrg, peersOrg) {
	return func(_ string) bool {
		return true
	}
}
```

否则返回不是本组织

```golang
return func(item string) bool {
	pkiID := common.PKIidType(item)
	msgsOrg := g.getOrgOfPeer(pkiID)
	if len(msgsOrg) == 0 {
		g.logger.Warning("Failed determining organization of", pkiID)
		return false
	}
	// Don't gossip identities of dead peers or of peers
	// without external endpoints, to peers of foreign organizations.
	if !g.hasExternalEndpoint(pkiID) {
		return false
	}
	// Peer from our org or identity from our org or identity from peer's org
	return bytes.Equal(msgsOrg, g.selfOrg) || bytes.Equal(msgsOrg, peersOrg)
}
```

### start方法

启动gossip服务

首先，创建goroutine调用g.syncDiscovery()同步节点发现；创建goroutine，调用g.handlePresumedDead()出来dead节点。

然后创建msgSelect函数，并根据此创建接受消息的通道，然后新启go routine，调用g.acceptMessages(incMsgs)处理消息

```golang
msgSelector := func(msg interface{}) bool {
	gMsg, isGossipMsg := msg.(proto.ReceivedMessage)
	if !isGossipMsg {
		return false
	}
	isConn := gMsg.GetGossipMessage().GetConn() != nil
	isEmpty := gMsg.GetGossipMessage().GetEmpty() != nil
	return !(isConn || isEmpty)
}
incMsg := g.comm.Accept(msgSelector)
go g.acceptMessages(incMsgs)
```

### syncDiscovery方法

如果!g.toDie，每g.conf.PullInterval周期调用g.disc.InitiateSync(g.conf.PullPeerNum)同步成员信息

### toDie方法

返回g.stopFlag是否为1

### handlePresumedDead方法

处理dead节点。对于g.toDieChan中的信号，将信号发送到g.toDieChan中，返回；对于g.comm.PresumedDead()的deadEndpoint，将其发送到g.presumedDead通道

### acceptMessages方法

处理接受到的消息

如果是g.toDieChan中的信号，将其发送到g.toDieChan并返回；如果是incMsg中的消息，调用g.handleMessage(msg)处理消息。

### handleMessage方法

处理消息

首先调用g.validateMsg方法验证消息。

如果消息IsChannelRestriced，调用g.chanState.lookupChannelForMsg(m)，查找消息的通道，如果节点不再通道中，也依旧转发消息以防消息是个StateInfo消息；如果没有找到channel，则调用g.validateLeadershipMessage校验LeaderElection消息，如果校验通过，调用GossipChannel.HanleMessage处理消息。

如果是Discovery消息(alive消息、memReq消息、memRes消息)，如果是membershipRequst消息，检查自己的信息是否和发送者一样，然后调用g.forwardDiscoveryMsg(m)转发discovery消息。

如果是pull消息，调用g.certStore.handleMessage(m)处理消息

### validateMsg方法

校验消息的签名和tag。首先校验Tag。然后如果是alive消息，调用g.disSecAdap.ValidateAliveMsg()方法校验消息；如果是StateInfo消息，调用g.validateStateInfoMsg校验消息。

### validateStateInfoMsg方法

校验stateInfo消息

首先创建校验器，然后从g.idMapper.Get()方法根据msg.GetStateInfo().PkiId获取identity，最后调用msg.Verify(identity, verifier)校验消息

### isInMyorg方法

判断member是否是自己的组织。首先调用g.getOrgOfPeer(member.PKIid)获得成员的组织，然后判断bytes.Equal(g.selfOrg, org)

### getOrgOfPeer方法

获取节点的组织。首先调用g.idMapper.Get(PKIID)获得节点的身份，然后调用g.secAdvisor.OrgByPeerIdentity()获取组织

### validateLeadershipMessage方法

首先调用msg.GetLeadershipMsg().PkiId获取pkiid，然后从g.idMapper中获取对应的identity，然后调用msg的Verify方法校验消息和身份

```golang
msg.Verify(identity, func(peerIdentity []byte, signature, message []byte) error{
	return g.mcs.Verify(identity, signature, message)
})
```

### selectOnlyDiscoveryMessages方法

选择discovery消息，即alive消息，request消息和response消息。

### forwardDiscoveryMsg方法

转发消息。将msg.GetGossipMessage发送到g.discAdapter.incChan中

### periodicalIdentityValidationAndExpiration方法

周期性地检查失效节点和活跃节点的身份。

```golang
go g.periodicalIdentityValidation(func(identity api.PeerIdentityType) bool{
	return true
}, identityExpirationCheckInterval)

go g.periodicalIdentityValidation(fucn(identity api.PeerIdentityType) bool {
	return false
}, identityInactivityCheckInterval)
```

### periodicalIdentityValidation方法

对于g.toDieChan中的信号，将其发送到g.toDieChan通道中，并返回；如果time.Afer(interval)的信号，调用g.SuspectPeers(suspectFunc)方法

### SuspectPeers方法

是gossip实例校验suspected节点，并关闭到无效身份的连接

```golang
for _, pkiID := range g.certStore.listRevokedPeers(isSuspected) {
	g.comm.CloseConn(&comm.RemotePeer{PKIID: pkiID})
}
```

### connect2BootstrapPeers方法

连接到bootstrap节点

对于g.conf.BootstrapPeers中的endpoint, 创建identitier。

```golang
identifier := func() (*discovery.PeerIdentification, error) {
	remotePeerIdentity, err := g.comm.Handshake(&comm.RemotePeer{Endpoint: endpoint})
	...
	sameOrg := bytes.Equal(g.selfOrg, g.secAdvisor.OrgByPeerIdentity(remotePeerIdentity))
	if !sameOrg {
		return nil, fmt.Errorf("...")
	}
	pkiID := g.mcs.GetPKIidOfCert(remotePeerIdentity)
	if len(pkiID) == 0 {
		return nil, fmt,Errorf("...")
	}
	return &discovery.PeerIdentification{ID: pkiID, selfOrg: sameOrg}, nil
}
```

最后创建到bootstrap的连接

```golang
g.disc.Connect(discovery.NetworkMember{
	InternalEndpoint: endpoint,
	Endpoint: endpoint,
}, identifier)
```

### selfNetworkMember方法

返回自身的networkmember对象

```golang
self := discovery.NetworkMember{
	Endpoint: g.conf.ExternalEndpoint,
	PKIid: g.comm.GetPkiid(),
	Metadata: []byte{},
	InternalEndpoint: g.conf.InternalEndpoint,
}
if g.disc != nil {
	self.Metadata = g.disc.Self().Metadata
}
return self
```

### JoinChan方法

加入通道

参数：

- joinMsg api.JoinChannelMessage
- chainID common.ChainID

首先调用g.chanState.joinChannel(joinMsg, chainID)方法加入channel。

对于joinMsg.Members的每个org，调用g.learnAnchorPeers(org, joinMsg.AnchorPeersOf(org))获得anchor peer

### learnAnchorPeers方法

连接到anchorpeer

参数：

- orgOfAnchorPeers api.OrgIdentityType
- anchorPers []api.AnchorPeer

首先对于anchorPeers中的节点，首先校验host和port是否合法，并跳过自身。

如果peer不是自己组织的且自己的endpoint为空，则continue。

创建identifier

```golang
identifier := func() (*discovery.PeerIdentificaiton, error){
	remotePeerIdentity, err := g.comm.Handshake(&comm.RemotePeer{Endpoint: endpoint})
	...
	isAnchorPeerInMyOrg := bytes.Equal(g.selfOrg, g.secAdvisor.OrgByPeerIdentity(remotePeerIdentity))
	if bytes.Equal(orgOfAnchorPeers, g.selfOrg) && !isAnchorPeerInMyOrg{
		//不是自己的组织，但其声称是
		return nil, errors.New(err)
	}
	pkiID := g.mcs.GetPKIidOfCert(remotePeerIdentity)
	...
	return &discovery.PeerIdentification{
		ID: pkiID,
		SelfOrg: isAnchorPeerInMyOrg
	}, nil

}
```

最后连接到anchor peer

```golang
g.disc.Connect(discovery.NetworkMember{
	InternalEndpoint: endpoint,
	Endpoint: endpoint,
}, identifier)
```

### sendGossipBatch方法

发送gossip的batch消息

参数：

- a []interface{}

创建要发送的消息，并调用g.gossipBatch(msgsGossip)发送

### gossipBatch方法

决定发送消息到那些节点。

为了效率，将拥有相同路由策略的消息一起发送，然后发送下一组消息。例如，发送所有通道C的区块当同一组peer，发送所有StateInfo消息到同一组peer。当发送区块时，只发送到那些在通道中广播了自己的节点；当发送StateInfo消息是，发送到通道中的节点；当发送标记着只能被在组织内发送，那么发送所有消息到同一组peer。其他没有限制的消息，发送到任意节点组。

参数：

- msgs []*proto.SignedGossipMessage

首先创建消息判断函数

```golang
isABlock := func(o insterface{}) bool {
	return o.(*proto.SignedGossipMessage).IsDataMsg()
}
isAStateInfoMsg := func(o interface{}) bool {
	return 0.(*proto.SignedGossipMessage).IsStateInfoMsg()
}
aliveMsgsWithNoEndpointAndInOurOrg := func(o interface{}) bool {
	msg := o.(*proto.SignedGossipMessage)
	if !msg.IsAliveMsg{
		return false
	}
	member := msg.GetAliveMsg().Membership
	return member.Endpoint == "" && g.isInMyorg(discovery.NetworkMember{PKIid: member.PkiId})
}
isOrgRestricted := func(o interface{}) bool {
	return aliveMsgsWithNoEndpointAndInOurOrg(o) || o.(*proto.SignedGossipMessage).IsOrgRestricted()
}
isLeadershipMsg := func(o interface{}) bool {
	return o.(*proto.SignedGossipMessage).IsLeadershipMsg()
}
```

调用partitionMessages方法对消息进行分类。

- gossip区块

```golang
blocks, msgs = partitionMessages(isABlock, msgs)
g.gossipInChan(blocks, func(gc channel.GossipChannel) filter.RoutingFilter {
	return filter.CombineRoutingFilters(gc.EligibleForChannel, gc.IsMemberInChan, g.isInMyorg)
})
```

- gossip Leadership消息

```golang
leadershipMsgs, msgs = partitionMessages(isLeadershipMsg, msgs)
g.gossipInChan(leadershipMsgs, func(gc channel.GossipChannel) filter.RoutingFilter {
	return filter.CombineRoutingFilters(gc.EligibleForChannel, gc.IsMemberInChan, g.isInMyorg)
})
```

- gossip stateInfo消息

```golang
stateInfoMsgs, msgs = partitionMessages(isAStateInfoMsg, msgs)
for _, stateInfMsg := range stateInfoMsgs {
	peerSelector := g.isInMyorg
	gc := g.chanState.lookupChannelForGossipMsg(stateInfMsg.GossipMessage)
	if gc != nil && g.hasExternalEndpoint(stateInfMsg.GossipMessage.GetStateInfo().PkiId) {
		peerSelector = gc.IsMemberInChan
	}
	peers2Send := filter.SelectPeers(g.conf.PropagatePeerNum, g.disc.GetMembership(), peerSelector)
	g.comm.Send(stateInfMsg, peers2Send...)
}
```

- gossip限制组织的消息

```golang
orgMsgs, msgs = partitionMessages(isOrgRestricted, msgs)
peers2Send := filter.SelectPeers(g.conf.PropagatePeerNum, g.disc.GetMembership(), g.isInMyorg)
for _, msg := range orgMsgs {
	g.comm.Send(msg, peers2Send...)
}
```

- gossip其他消息

```golang
for _, msg := range msgs {
	if !msg.IsAliveMsg() {
		g.logger.Error("Unknown message type", msg)
		continue
	}
	selectByOriginOrg := g.peersByOriginOrgPolicy(discovery.NetworkMember{PKIid: msg.GetAliveMsg().Membership.PkiId})
	peers2Send := filter.SelectPeers(g.conf.PropagatePeerNum, g.disc.GetMembership(), selectByOriginOrg)
	g.sendAndFilterSecrets(msg, peers2Send...)
}
```

### partitionMessages方法

区分消息

参数：

- pred common.MessageAcceptor
- a []*proto.SignedGossipMessage

返回值：

- []*proto.SignedGossipMessage
- []*proto.SignedGossipMessage

如果满足pred条件，添加到s1中；否则，添加到s2中。返回s1和s2

### gossipInChan方法

根据channel的routing policy发送Gossip消息块

参数：

- messages []*proto.SignedGossipMessage
- chanRoutingFactory channelRoutingFilterFactory

调用extractChannels(messages)从消息中获取所有的channel，然后针对每个channel将消息进行区分，调用g.comm.Send方法发送消息

### extractChannels方法

从消息中获取所有channel

针对每个消息，获取其channel，收集到一起

### hasExternalEndpoint方法

返回g.disc.Lookup(PKIID).Endpoint != ""

### peersByOriginOrgPolicy方法

首先通过peer.PKIid获取peersOrg，如果是自己的组织，返回filter.SelectAllPolicy。否则，返回选择来源组织的节点和自己组织的节点

```golang
return func(member discovery.NetworkMember) bool {
	memberOrg := g.getOrgOfPeer(member.PKIid)
	if len(memberOrg) == 0 {
		return false
	}
	isFromMyOrg := bytes.Equal(g.selfOrg, memberOrg)
	return isFromMyOrg || bytes.Equal(memberOrg, peersOrg)
}
```

### sendAndFilterSecrets方法

对于peer中的每个消息，不要转发外部组织的alive消息到没有外部endpoint的节点。将msg.Envelope.SecretEnvelope设置为nil，调用g.comm.Send发送消息。

### Gossip方法

gossip消息

参数：

- msg *proto.GossipMessage

如果是数据消息，不对消息进行签名；否则，调用g.mcs.Sign(msg)对消息进行签名

如果消息IsChannelRestricted，调用g.chanState.getGossipChannelByChainID(msg.Channel)获取通道，如果msg是数据消息，调用gc.AddToMsgStore将其添加到消息存储。

如果，g.conf.PropagateIterations为0，直接返回。否则，调用g.emitter.Add(sMsg)发送消息

### Send方法

发送消息到远端节点

参数：

- msg *proto.GossipMessage
- peers ...*comm.RemotePeer

调用g.comm.Send(m, peers...)发送消息

### Peers方法

返回discovery.NetworkMember的列表

### PeersOfChannel方法

返回存活的且是订阅到给定通道的节点

### Stop方法

停止gossip组件

### UpdateMetadata方法

调用g.disc.UpdateMetadata(md)更新元信息

### UpdateChannelMetadata方法

更新分发到其他节点的、与通道相关的状态的节点自身的元信息

参数：

- md []byte
- chainID commom.ChainID

首先，调用g.chanState.getGossipChannelByChainID获取通道，然后调用g.createStateInfoMsg创建StateInfo消息，返回调用gc.UpdateStateInfo(stateInfMsg)更新元信息

### Accept方法

返回一个专用的只读通道，传输那些由其他节点发来的、满足特定条件的消息。

参数：

- acceptor commom.MessageAcceptor
- passThrough bool

返回值：

- <-chan *proto.GossipMessage
- <-chan proto.ReceivedMessage

如果passThrough为真，gossip层不做处理，直接返回nil, g.comm.Acceptor.

否则，创建根据类型接受消息函数

```golang
acceptByType := func(o interface{}) bool {
	if o, isGossipMsg := o.(*proto.GossipMessage); isGossipMsg {
		return acceptor(o)
	}
	if o, isSignedMsg := o.(*proto.SignedGossipMessage); isSignedMsg {
		sMsg := o
		return acceptor(sMsg.GossipMessage)
	}
	return false
}
```

将aceptByType注册到channel中，返回通道inCh；创建outCh传送proto.GossipMessage。

新建go routine，处理信号。如果是g.toDieChan的信号，将其发送到g.toDieChan中，并返回；如果是inCh中的信号，将其转换为GossipMessage发送到outCh中。返回outCh,nil

### createStateInfoMsg方法

创建stateInfo消息

参数：

- metadata []byte
- chainID common.ChainID

返回值：

- *proto.SignedGossipMessage
- error

### disclosurePolicy方法

关闭策略

参数：

- remotePeer *discovery.NetworkMember

返回值：

- discovery.Sieve
- discovery.EnvlopeFilter

首先根据remotePeer.PKIid调用g.getOrgOfPeer获取remotePeerOrg。如果remotePeerOrg为空，返回

```golang
if len(remotePeerOrg) == 0 {
	g.logger.Warning("Cannot determine organization of", remotePeer)
	return func(msg *proto.SignedGossipMessage) bool {
			return false
		}, func(msg *proto.SignedGossipMessage) *proto.Envelope {
			return msg.Envelope
		}
}
```

否则，创建Sieve函数和EnvelopeFilter函数。

```golang
return func(msg *proto.SignedGossipMessage) bool {
	if !msg.IsAliveMsg() {
		g.logger.Panic("Programming error, this should be used only on alive messages")
	}
	org := g.getOrgOfPeer(msg.GetAliveMsg().Membership.PkiId)
		if len(org) == 0 {
			g.logger.Warning("Unable to determine org of message", msg.GossipMessage)
			// Don't disseminate messages who's origin org is unknown
			return false
		}

		// Target org and the message are from the same org
		fromSameForeignOrg := bytes.Equal(remotePeerOrg, org)
		// The message is from my org
		fromMyOrg := bytes.Equal(g.selfOrg, org)
		// Forward to target org only messages from our org, or from the target org itself.
		if !(fromSameForeignOrg || fromMyOrg) {
			return false
		}

		// Pass the alive message only if the alive message is in the same org as the remote peer
		// or the message has an external endpoint, and the remote peer also has one
		return bytes.Equal(org, remotePeerOrg) || msg.GetAliveMsg().Membership.Endpoint != "" && remotePeer.Endpoint != ""
	}, func(msg *proto.SignedGossipMessage) *proto.Envelope {
		if !bytes.Equal(g.selfOrg, remotePeerOrg) {
			msg.SecretEnvelope = nil
		}
		return msg.Envelope
	}
```

### peerByOriginOrgPolicy
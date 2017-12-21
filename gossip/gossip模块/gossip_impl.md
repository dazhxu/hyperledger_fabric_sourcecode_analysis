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

首先调用msg.GetLeadershipMsg().PkiId获取pkiid，然后
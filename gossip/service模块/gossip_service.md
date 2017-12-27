gossip_service
===

gossip_service定义了GossipService接口

# 变量与方法定义

## 变量

### gossipService实例

```golang
var (
	gossipServiceInstance *gossipServiceImpl
	once sync.Once
)
```

### gossip实例

```golang
type gossipSvc gossip.Gossip
```

### 日志

```golang
var logger = util.GetLogger(util.LoggingServiceModule, "")
```

# DeliveryServiceFactory接口

DeliveryServiceFactory接口是创建和初始化delivery服务实例的工厂类

```golang
type DeliveryServiceFactory interface {
	Service(g GossipService, endpoints []string, msc api.MessageCryptoService) (deliverclient.DeliverService, error)
}
```

# deliveryFactoryImpl结构

deliveryFactoryImpl是对DeliveryServiceFactory接口的实现

## Service方法

创建一个deliverclient.DeliverySerivce示例。

```golang
return deliverclient.NewDeliverService(&delivercli.Config{
	CryptoSvc: mcs,
	Gossip: g,
	Endpoints: endpoints,
	ConnFactory: deliverclient.DefaultConnectionFactory,
	ABCFactory: deliverclient.DefaultABCFactory,
})
```

# joinChannelMessage结构

joinChannelMessage结构是对api.JoinChannelMessage的实现

## 属性

包含序列化和anchorPeer属性

```golang
type joinChannelMessage struct {
	seqNum 					uint64
	members2AnchorPeers 	map[string][]api.AnchorPeer
}
```

## 方法

### SequenceNumber方法

返回jcm.seqNum

### Members方法

返回jcm的成员

### AnchorPeerOf

返回特定组织的anchorPeer

# GossipService接口

GossipService将gossip和state功能封装到一个接口中

```golang
type GossipService interface {
	gossip.Gossip

	NewConfigEventer() ConfigProcessor
	InitializeChannel(chainID string, commiter committer.Committer, endpoints []string)
	GetBlock(chainID string, index uint64) *common.Block
	AddPayload(chainID string, payload *proto.Payload) error
}
```

- NewConfigEventer方法：创建一个ConfigProcessor，configtx.Manager能把配置更新路又道这个ConfigProcessor
- InitializeChannel方法：InitializeChannel分配state提供者，并且每个channel每次执行都应该被调用
- GetBlock方法：返回给定chain的区块
- AddPayload方法：添加消息payload到给定的chain

# gossipServiceImpl结构

gossipServiceImpl是对GossipService的实现

## 属性

gossipServiceImpl包含gossip.Gossip，state.GossipStateProvider，election.LeaderElectionService，deliveryService等属性

```golang
type gossipServiceImpl struct {
	gossipSvc
	chains 				map[string]state.GossipStateProvider
	deliveryElection	map[string]election.LeaderElectionService
	deliveryService 	deliverclient.DeliverFactory
	deliveryFactory 	DeliveryServiceFactory
	lock 				sync.RWMutex
	idMapper 			identity.Mapper
	mcs 				api.MessageCryptoService
	peerIdentity 		[]byte
	secAdv  			api.SecurityAdvisor
}
```

## 构造方法

### InitGossipService方法

初始化gossip服务

参数：

- peerIdentity []byte
- endpoint string
- s *grpc.Server
- mcs api.MessageCryptoService
- secAdv api.SecurityAdvisor
- secureDialOpts api.PeerSecureDialOpts
- bootPeers ...string

返回值：

- error

调用InitGossipServiceCustomDeliveryFactory方法初始化gossip服务

### InitGossipServiceCustomDeliveryFactory方法

初始化带自定义的delivery工厂接口实现的gossip服务。

### GetGossipService返回gossip服务实例

返回gossipServiceInstance

### NewConfigEventer方法

调用newConfigEventer创建一个configProcessor

### InitializeChannel方法

分配state provider，并在每个channel每次执行时被调用

参数：

- chainID string
- committer committer.Committer
- endpoints []string

首先为给定的committer创建新的state provider。如果g.deliveryService为空，调用g.deliveryFactory.Service创建deliveryService/

```golang
g.chains[chainID] = state.NewGossipStateProvider(chainID, g, committer, g.mcs)
if g.deliveryService == nil {
	g.deliveryService, err = g.deliveryFactory.Service(gossipServiceInstance, endpoints, g.mcs)
}
```

如果g.deliveryService不为空。首先获取leaderElection选项和isStaticOrgLeader选项，如果都为真，产生一个panic；如果leaderElection打开，将g.leaderElection[chainID] = g.newLeaderElectionComponent(chainID, g.onStatusChangeFactory(chainID, committer))选择leader；如果isStaticOrgLeader打开，调用g.deliveryService.StartDeliveryForChannel(chainID, committer, func() {})将此节点连接到service。

### configUpdated方法

构建joinChannelMessage消息，并发送给gossipSvc

首先，获取string(g.secAdv.OrgByPeerIdentity(api.PeerIdentityType(g.peerIdentity)))获取自己的组织。如果!g.amIinChannel(myOrg, config)，说明本身节点不再这个channel中，直接返回。

否则，构建joinChannelMessage消息

```golang
jcm := &joinChannelMessage{seqNum: config.Sequence(), members2AnchorPeers: map[string][]api.AnchorPeer{}}
```

对于config中的组织，将每个组织的anchorPeers添加到jcm.members2AchorPeers[appOrg.MSPID()]中。

最后调用g.JoinChan(jcm, gossipCommon.ChainID(config.ChainID()))发消息更新通道

### GetBlock方法

调用g.chains[chainID].GetBlock(index)获取给定chain的区块

### AddPayload方法

调用g.chains[chainID].AddPayload(payload)添加消息payload到给定chain

### Stop方法

停止gossip模块

首先对于g.chains中的通道，调用ch.Stop()停止；

对于g.leaderElection中的electionService，调用electionService.Stop方法停止

调用g.gossipSvc.Stop停止g.gossipSvc。

如果g.deliveryService != nil, 调用g.deliveryService.Stop()停止

### newLeaderElectionComponent方法

通过g.peerIdentity获取PKIid，并调用election.NewAdapter创建adapter，最后调用election.NewLeaderElectionService创建leader选举实例

### amIinChannel方法

对于orgListFormConfig中的组织，判断是否和myOrg相同

### onStatusChangeFactory方法

返回一个函数。

```golang
return func(isLeader bool) {
	if isLeader {
		yield := func() {
			g.lock.RLock()
			le := g.leaderElection[chainID]
			g.lock.RUnlock()
			le.Yield()
		}
		if err := g.deliveryService.StartDeliverForChannel(chainID, committer, yield); err!=nil {
			logger.Error(...) 
		}
	} else {
		if err := g.deliveryService.StopDeliverForChannel(chainID); err != nil {
			logger.Error(...)
		}
	}
}
```

### orgListFromConfig方法

从config中获取org列表
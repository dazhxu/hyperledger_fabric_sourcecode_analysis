gossip
===

gossip定义了Gossip接口和Config结构

# Gossip接口

Gossip接口定义了gossip模块的接口

```golang
type Gossip interface {
	Send(msg *proto.GossipMessage, peers ...*comm.RemotePeer)
	Peers() []dicovery.NetworkMember
	PeersOfChannel(common.ChainID) []discovery.NetworkMember
	UpdateMetadata(metadata []byte)
	UpdateChannelMetadata(metadata []byte, chainID common.ChainID)
	Gossip(msg *proto.GossipMessage)
	Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <-chan proto.ReceivedMessage)
	JoinChan(joinMsg api.JoinChannelMessage, chainID common.ChainID)
	SuspectPeers(s api.PeerSuspector)
	Stop()
}
```

- Send方法：发送消息到远端节点
- Peers方法：返回认为是存活的节点
- PeersOfChannel方法：返回订阅到给定channel的且被认为是存活的节点
- UpdateMetadata方法：更新dicovery层的节点自身的metadata，这些信息分发到其他节点
- UpdateChannelMetadata方法：更新节点分发到其他节点的与channel相关的metadata
- Gossip方法：向网络中的其他节点gossip消息
- Accept方法：返回一个专用的通道，为传送其他节点发送的满足特定条件的消息。如果passThrough为false，message有gossip层预先处理；否则，gossip层不做处理，发送反馈消息给发送者
- JoinChan方法：使gossip实例加入一个channel
- SuspectPeers方法：使gossip实例校验suspected节点身份，如果身份无效，关闭与其的所有连接
- Stop方法：停止gossip模块

# Config结构

Config结构为gossip模块的配置

```golang
type Config struct {
	BindPort            int      // 绑定的端口，只用作测试
	ID                  string   // 实例的ID
	BootstrapPeers      []string // 在启动时可以连接的ID
	PropagateIterations int      // 一个消息发送到其他节点的次数
	PropagatePeerNum    int      // 发送消息到节点的个数

	MaxBlockCountToStore int // 在memory中存储的区块的最大数量

	MaxPropagationBurstSize    int           // 在发送到其他前存储消息的最大数量
	MaxPropagationBurstLatency time.Duration // 在发送到其他节点前消息存储的时间

	PullInterval time.Duration // 决定了pull阶段的频率
	PullPeerNum  int           // 从多少个节点拉数据

	SkipBlockVerification bool // 是否应该跳过验证区块

	PublishCertPeriod        time.Duration    // 包含在Alive消息中的启动证书的时间
	PublishStateInfoInterval time.Duration    // 决定了发送state info到其他节点的频率
	RequestStateInfoInterval time.Duration    // 决定了从其他节点拉state info的频率
	TLSServerCert            *tls.Certificate // 节点的TLS证书

	InternalEndpoint string // 发布给组织内节点的endpoint
	ExternalEndpoint string // 发布给其他组织的endpoint
}
```
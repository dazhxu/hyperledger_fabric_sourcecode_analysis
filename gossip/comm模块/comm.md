comm
===

comm文件中定义了peer之间进行通信的Comm接口，包含获取PKIid、发送、探测、握手、接收、假定断开、关闭连接、停止等功能，其实现在comm_impl.go文件中。

```golang
type Comm interface {
	GetPKIid() common.PKIidType
	Send(msg *proto.SignedGossipMessage, peers ...*RemotePeer)
	Probe(peer *RemotePeer) error
	Handshake(peer *RemotePeer) error
	Accept(common.MessageAcceptor) <-chan proto.ReceivedMessage
	PresumedDead() <-chan common.PKIidType
	CloseConn(peer *RemotePeer)
	Stop()
}
```

comm_impl定义了Comm接口的实现对象commImpl。commImpl对象维护了节点身份、idMapper、连接选项、连接存储、通道多路选择器、监听器、grpc服务器、订阅等属性。

```golang
type commImpl struct {
	selfCertHash	[]byte
	peerIdentity	api.PeerIdentity
	idMapper        identity.Mapper
	logger			*logging.Logger
	opts			[]grpc.DialOption
	secureDialOpts	func() []grpc.DialOption
	connStore		*connectionStore
	PKIID 			[]byte
	deadEndpoints	chan common.PKIidType
	msgPublisher	*ChannelDeMultiplexer
	lock			*sync.RWMutex
	lsnr			net.Listener
	gSrv			*grpc.WaitGroup
	subscriptions	[]chan proto.ReceivedMessage
	port 			int
	stopping 		int32
}
```


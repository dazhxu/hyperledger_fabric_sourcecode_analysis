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
	gSrv			*grpc.Server
	exitChan		chan struct{}
	stopWG 			*grpc.WaitGroup 
	subscriptions	[]chan proto.ReceivedMessage
	port 			int
	stopping 		int32
}
```

# commImpl属性介绍

## 身份相关属性

- selfCertHash: 指的是节点tls证书的SHA256哈希

- peerIdentity: 指节点的证书

- PKIID：指节点的pki-id

- idMapper: 存储了pki-id到证书的映射

## 连接相关属性

- opts：grpc连接选项

- secureDialOpts: grpc安全连接选项

- connStore：存储了与此节点进行的连接，即存储了从pki-id到连接的映射

- deadEndpoints: 断开的节点的一个通道

- lsnr：peer节点的监听器

- gSrc：grpc服务

- port：端口

## 多通道相关属性

- msgPublisher：通道的多路选择器，将消息路由到正确的通道中

- subscriptions：传输向channel订阅的消息的通道

# 构造方法及相关方法

## NewCommInstance方法

NewCommInstance新建一个comm实例，并把自己绑定到给定的gRPC服务器上

参数：
- s *grpc.Server: 绑定的gRPC服务器
- cert tls.Certificate: tls证书
- idStore identity.Mapper: 

## NewCommInstanceWithServer

- 


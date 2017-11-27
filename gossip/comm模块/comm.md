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

# 构造及相关方法

## NewCommInstance方法

NewCommInstance新建一个comm实例，并把自己绑定到给定的gRPC服务器上

参数：

- s *grpc.Server: 绑定的gRPC服务器
- cert tls.Certificate: tls证书
- idStore identity.Mapper: 存储pki-id到identity的映射
- peerIdentity api.PeerIdentity: peer的身份
- secureDialOpts api.PeerSecureDialOpts: 返回一个grpc.DialOpts的函数
- dialOpts ...grpc.DialOption: grpc的连接选项

返回值：

- Comm：Comm实例
- error

NewCommInstance方法首先从获取peer.gossip.dialTimeout参数，并添加到dialOpts对象中；

然后调用NewCommInstanceWithServer方法创建一个Comm实例；

如果tlscert不为空且有证书，将Comm实例的selfCertHash设置为cert.Certificate[0]的SHA256哈希；

最后将Comm实例绑定到grpc服务器上

```golang
proto.RegisterGossipServer(s, commInst.(*commImpl))
```

## NewCommInstanceWithServer方法

创建一个Comm实例

如果dialOpts的长度为0，将peer.gossip.dialTimeout参数添加到dialOpts；

如果port大于0，调用createGRPCLayer方法，创建一个grpc服务器；

创建一个commImpl对象；

调用newConnStore方法创建connStore对象，添加到commImpl对象的属性中；

如果port大于0，将commImpl对象的stopWG等待组加1。新建线程启动服务器，线程结束时，stopWG减1。在原来线程调用proto.RegisterGossipServer(s, commInst)将comm对象绑定到grpc服务器上。

## createGRPCLayer方法

createGRPCLayer方法创建grpc服务器。

参数：

- port int：监听地址

返回值：

- *grpc.Server: grpc服务器
- net.Listener: 监听器
- api.PeerSecureDialOpts: 节点的安全连接选项
- []byte: tlscert哈希

首先，调用本模块的crypto.go文件中的GenerateCertificatesOrPanic方法获取tls.Certificate，并将证书进行SHA256哈希，得到certHash；

初始化tlsConf，并将其添加到serverOpts和dialOpts

```golang
tlsConf := &tls.Config{
	Certificates: []tls.Certificate{cert},
	ClientAuth: tls.RequestClientCert,
	InsecureSkipVerify: true,
}
serverOpts = append(serverOpts, grpc.Creds(credentials.NewTLS(tlsConf)))
```

```golang
ta := credentials.NewTLS(&tls.Config{
	Certificates: []tls.Certificate{cert},
	InsecureSkipVerify: true,
})
dialOpts = append(dialOpts, grpc.WithTransportCredentials(ta))
```

然后监听tcp端口，ll, err = net.Listen("tcp", listenAddress)

最后创建grpc服务器，s = grpc.NewServer(serverOpts...)

# commImpl对象的普通方法

## createConnection方法

创建一个到远端节点的connection连接。

参数：

- endpoint string: 远端节点地址
- expectedPKIID comm.PKIidType: 期望的PKIID

返回值：

- *connection：到远端节点的连接
- error

首先，设置连接选项；

```golang
dialOpts = append(dialOpts, c.secureDialOpts()...)
dialOpts = append(dialOpts, grpc.WithBlock())
dialOpts = append(dialOpts, c.opts)
```

然后，调用grpc的Dial方法连接，然后根据连接创建一个gossipClient，调用Ping方法测试连通性；

```golang
cc, err = grpc.Dial(endpoint, dialOpts...)
cl := proto.NewGossipClient(cc)
if _, err = cl.Ping(context.Backgroud(), &proto.Empty{}); err != nil {
	cc.Close()
	return nil, err
}
```

获取一个cancel context和cancel function；

```golang
ctx, cf := context.WithCancel(context.Backgroud())
```

调用GossipStream方法获取ctx的数据流，创建一个connection：(1) 调用commImpl对象的authenticateRemotePeer方法认证远端节点；(2) 调用newConnection方法新建connection对象; (3) 设置connection对象的属性(pkiID、info、cancel、handle)，其中handle方法是将gossip消息交给commImpl对象的msgPublisher属性的Demultiplex方法处理，并返回

## authenticateRemotePeer方法

认证远端节点，返回ConnectionInfo对象

参数：

- stream stream: 数据流

返回值：

- *proto.ConnectionInfo
- error

首先，从stream中创建ctx，并获取远端节点的地址和certHash；

设置签名器，如果TLS选项打开，则使用idMapper.Sign作为签名器；否则，不签名；

创建ConnectionMsg，并发送；

```golang
cMsg, err = c.createConnectionMsg(c.PKIID, c.selfCertHash, c.peerIdentity, signer)
stream.Send(cMsg.Envelope)
```

从stream中读取回应，将远端节点的pkiID和Identity的映射放在idMapper中；

```golang
m, err := readWithTimeout(stream, util.GetDurationOrDefault("peer.gossip.connTimeout", defConnTimeout), remoteAddress)
receivedMsg := m.GetConn()
err = c.idMapper.Put(receivedMsg.PkiId, receivedMsg.Identity)
```

创建connInfo,如果tls选项打开，则调用SignedGossipMessage对象的Verify方法验证远端peer

```golang
connInfo := &proto.ConnectionInfo{
	ID: 		receivedMsg.PkiId,
	Identity:	receivedMsg.Identity,
	Endpoint:	remoteAddress,
}
```
## Send方法

发送消息到远端节点

参数：

- msg *proto.SignedGossipMessage
- peers ...*RemotePeer

对于每一个peer，启动一个线程，调用sendToEndpoint方法，将消息发送过去。

## sendToEndpoint方法

发送消息到某个远端节点

参数：

- peer *RemotePeer
- msg *proto.SignedGossipMessage

首先从commImpl对象的connStore中获取peer的连接。定义disConnectOnErr函数，当出现错误是断开连接。然后调用connection对象的send方法将消息发到远端节点。

## Probe方法

探测远端节点。

参数：

- remotePeer *RemotePeer

首先，设置连接选项；

```golang
dialOpts = append(dialOpts, c.secureDialOpts()...)
dialOpts = append(dialOpts, grpc.WithBlock())
dialOpts = append(dialOpts, c.opts...)
```

然后，向远端节点进连接，并根据连接创建gossip客户端，然后调用Ping方法探测远端节点。

```golang
cc, err := grpc.Dial(remotePeer.Endpoint, dialOpts...)
```

## Handshake方法

与远端节点进行握手，完成认证。

参数：

- remotePeer *RemotePeer

返回值：

- api.PeerIdentityType
- error

首先，设置连接选项；然后调用grpc.Dial方法连接远端节点，根据连接，调用proto.NewGossipClient方法获取gossipClient，然后调用gossipClient的GossipStream方法，然后调用authenticateRemotePeer认证点多，并获取连接信息connInfo。然后比对remotePeer的PKIid和connInfo的ID是否相同，如果相同就返回connInfo的Identity。

## Accept方法

接受消息，返回一个专用的通道。

参数：

- acceptor common.MessageAcceptor: 用来决定创建MessageAcceptor实例的subscriber关心那些消息。

返回值：

- <-chan proto.ReceivedMessage

首先将acceptor添加到多路选择器上，并返回一个通用的通道。创建一个ReceivedMessage的专用的通道specificChan，并将器添加到commImpl对象的subscriptions中。

```golang
genericChan := c.msgPublisher.AddChannel(acceptor)
specificChan := make(chan proto.ReceivedMessage, 10)
...
c.lock.Lock()
c.subscriptions = append(c.subscriptions, specificChan)
```

创建一个新的线程处理消息：对于从genericChan的消息，将消息转换成ReceivedMessageImpl对象，发送到specificChan对象。

```golang
go func() {
	defer func() {
		recover()
	} ()

	c.stopWG.Add(1)
	defer c.stopWG.Done

	for {
		select {
			case msg := <-genericChan:
				specificChan <- msg.(*ReceivedMessageImpl)
			case s := <-c.exitChan:
				c.exitChan <- s
				return
		}
	}
}()

return specificChan
```

## PresumedDead方法

返回c.deadEndpoints

## CloseConn方法

调用c.connStore.closeConn方法断开与peer的连接

## emptySubscriptions方法

清空c.subscriptions中的所有通道

## Stop方法

依次停止gSrv、lsnr、connStore、msgPublisher，调用emptySubscription清空通道，最后等待stopWG。

## GossipStream方法


参数：

- stream proto.Gossip_GossipStreamServer

首先调用authenticateRemotePeer(stream)认证节点并获取connInfo；然后调用c.connStore.OnConnected(stream,connInfo)获取连接，如果连接为空，说明已经存在了到此远端接的连接，关闭这个stream；创建handler函数，将收到的消息发送到Demultiplex处理，并将conn.handler设置为处理函数；然后调用conn.ServiceConnection()方法启动服务。

## Ping方法

返回一个空的对象

参数：

- context.Context
- *proto.Empty

返回值：

- *proto.Empty
- error

```golang
func (c *commImpl) Ping(context.Context, *proto.Empty) (*proto.Empty, error) {
	return &proto.Empty{}, nil
}
```

## disconnect方法

端口与某个peer的连接

参数：

- pkiID common.PKIidType

将pkiID发送到c.deadEndpoint通道，调用c.connStore.closeByPKIid(pkiID)关闭连接

## createConnectionMsg方法

创建一个连接信息

参数：

- pkiID common.PKIidType：远端peer的pkiID
- certHash []byte：tls证书的哈希
- cert api.PeerIdentityType：tls证书
- signer proto.Signer：签名器

```golang
m := &proto.GossipMessage{
	Tag: proto.GossipMessage_EMPTY,
	Nonce: 0,
	Content: &proto.GossipMessage_Conn {
		Conn: &proto.ConnEstablish {
			TlsCertHash: certHash,
			Identity: cert,
			PkiId: pkiID,
		},
	},
}
sMsg := &proto.SignedGossipMessage {
	GossipMessage: m,
}
_, err := sMsg.Sign(signer)
return sMsg, err
```

# 其他方法和接口

## readWithTimeout方法

从stream中读消息

参数：

- stream interface{}
- timeout time.Duration
- address string

返回值：

- *proto.SignedGossipMessage
- error

首先，创建两个通道，用于线程间通信，一个用来传递gossip消息，一个用来传递错误信息

```golang
incChan := make(chan *proto.SignedGossipMessage, 1)
errChan := make(chan error, 1)
```

新建一个线程，用于接受消息。如果是服务器，调用服务器的接受方法；如果是客户端，调用客户端的接受方法。将接受到的消息发送到incChan通道中

```golang
go func() {
	if srvStr, isServerStr := stream.(proto.Gossip_GossipStreamServer); isServerStr {
		if m, err := srvStr.Recv(); err == nil {
			msg, err := m.ToGossipMessage()
			if err != nil {
				errChan <- err
				return
			}
			incChan <- msg
		}
	} else if clStr, isClientStr := stream.(proto.Gossip_GossipStrStreamClient); isClientStr {
		if m, err := clStr.Recv(); err == nil {
			msg, err := m.ToGossipMessage()
			if err != nil {
				errChan <= err
				return
			}
			incChan <- msg
		}
	} else {
		panic(fmt.Errorf("Stream isn't a GossipStreamServer or a GossipClient, but %v. Aborting", reflect.TypeOf(stream)))
	}
}()
```

利用select选择器，对消息进行处理。如果超时，返回超时；如果是incChan的消息，返回消息；如果是errChan消息，返回err

```golang
select {
	case <- time.NewTicker(timeout).C:
		return nil, fmt.Errorf("Timed out waiting for connection message from %s", address)
	case m := <-incChan:
		return m, nil
	case err := <-errChan:
		return nil, err
}
```

## stream接口

继承grpc.Stream, 包含Send方法和Recv方法

```golang
type stream interface {
	Send(envelope *proto.Envelope) error
	Recv() (*proto.Envelope, error)
	grpc.Stream
}
```


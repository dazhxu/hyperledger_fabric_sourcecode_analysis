conn
===

conn子模块主要定义了connection对象和connectionStore对象，connection指与远端节点的连接，connectionStore存储了到远端节点的所有连接。

# msgSending对象

定义了发送的数据。

```golang
type msgSending struct {
	envelope *proto.Envelope
	onErr func(error)
}
```

# connection对象

## 属性

connection对象包含了缓存通道、节点身份、连接、客户端stream、服务器stream等属性

```golang
type connection struct {
	cancel			context.CancelFunc
	info			*proto.ConnectionInfo
	outBuff			chan *msgSending
	logger			*logging.Logger
	pkiID 			common.PKIidType
	handler			handler
	conn 			*grpc.ClientConn
	cl 				proto.GossipClient
	clientStream	proto.Gossip_GossipStreamClient
	serverStream 	proto.Gossip_GossipStreamServer
	stopFlag 		int32
	stopChan 		chan struct{}
	sync.RWMutex
}
```

其中：

- outBuff：peer的发送缓存
- pkiID：common.PKIidType
- handler: 接受到消息调用的方法
- conn：到远端节点的gRPC连接
- cl: 远端节点的grpc stub
- clientStream：到远端节点的客户端侧的stream
- serverStream：到
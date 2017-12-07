state
===

state模块定义了GossipStateProvider接口及GossipAdapter接口，以及GossipStateProviderImpl类作为GossipStateProvider的实现

# 常量

```golang
const (
	defAntiEntropyInterval = 10*time.Second
	defAntiEntropyStateResponseTimeout = 3*time.Second
	defAntiEntropyBatchingSize = 10

	defChannelBufferSize = 100
	defAntiEntropyMaxRetries = 3

	defMaxBlockDistance = 100

	blocking = true
	nonBlocking = false

	enqueueRetryInterval = time.Millisecond*100
)
```

# GossipAdapter接口

GossipAdapter为state provider定义了gossip/communication需要的接口

```golang
type GossipAdapter interface {
	Send(msg *proto.GossipMessage, peers ...*comm.RemotePeer)
	Accept(acceptor common2.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <-chan proto.ReceivedMessage)
	UpdateChannelMetadata(metadata []byte, chainID common2.ChainID)
	PeersOfChannel(common2.ChainID) []discovery.NetworkMember
}
```

- Send方法：发送一个消息到远端节点
- Accept方法：返回一个消息的专用的通道，这些消息是由其他节点发送的并满足一定的预设条件。如果passThrough为false，消息被gossip层处理；否则，gossip层忽略消息消息是用来发送回复给发送者
- UpdateChannelMetadata：更细节点自身的关于channel的metadata
- PeersOfChannel：返回订阅了特定channel的、被认为是存活的NetworkMember

# GossipStateProvider接口

定义了获取缺失区块的接口

```golang
type GossipStateProvider interface {
	GetBlock(index uint64) *common.Block
	AddPayload(payload *proto.Payload) error
	Stop()
}
```

- GetBlock方法： 获取某个序号的区块
- AddPayload方法：添加payload
- Stop方法：停止state传送对象

# GossipStateProviderImpl类

处理账本需要的新区块的滑动窗口

## 属性

GossipStateProviderImpl定义了ChainID、GossipAdapter、commChan、committer、stateResponseCh、stateRequestCh等属性

```golang
type GossipStateProviderImpl struct {
	mcs api.MessageCryptoService
	chainID string
	gossip GossipAdapter
	gossipChan <-chan *proto.GossipMessage
	commChan <-chan proto.ReceivedMessage
	payloads PayloadsBuffer
	committer committer.Committer
	stateResponseCh chan proto.ReceivedMessage
	stateRequestCh chan proto.ReceivedMessage
	stopCh chan struct{}
	done sync.WaitGroup
	once sync.Once
	stateTransferActive int32
}
```

## 构造函数

### NewGossipStateProvider方法

创建一个gossip state provider

参数：

- chainID string: 链的名字
- g GossipAdapter: GossipAdapter接口
- committer committer.Committer：提交器
- mcs api.MessageCryptoService: 消息加密服务

返回值：

- GossipStateProvider

首先调用g.Accept方法为消息传送创建通道,只传送数据消息。

```golang
gossipChan, _ := g.Accept(func(message interface{}) bool {
	return message.(*proto.GossipMessage).IsDataMsg() && bytes.Equal(message.(*proto.GossipMessage).Channel, []byte(chainID))
}, false) 
```

定义remoteMsgFilter方法，认证nodeMetadate，然后构造commChan

```golang
remoteStateMsgFilter := func(message interface{}) bool {
	receivedMsg := message.(proto.ReceivedMessage)
	msg := receivedMsg.GetGossipMessage()
	if !msg.IsRemoteStateMessage() {
		return false
	}
	if !receivedMsg.GetConnectionInfo().IsAuthentical() {
		return true
	}
	connInfo := receivedMsg.GetConnectionInfo()
	authErr := mcs.VerifyByChannel(msg.Channel, connInfo.Identity, connInfo.Auth.Signature, connInfo.Auth.SignedData)
	if authErr != nil {
		...
		return false
	}
	return true
}

_, commChan := g.Accept(remoteStateMsgFilter, true)
```

创建GossipStateProviderImpl对象，更新ChannelMetadata

最后，新启线程，调用s.listen()监听到达的通讯；新启线程，调用s.deliverPayloads()发送排序好的payloads；新启线程，调用antiEntropy()同步区块；新启线程，调用processStateRequests()处理state请求

## 普通方法

### listen方法

对于gossipChan中的信号，新启线程，调用queueNewMessage方法，排队新消息；对于s.commChan中的信号，新启线程调用directMessage方法处理消息；对于stopCh中的消息，将空对象发送到stopCh中并返回。

### queueNewMessage方法

新消息通知/处理

参数：

- msg *proto.GossipMessage

调用msg.GetDataMsg获取消息数据，调用s.addPayload(dataMsg.GetPayload(), nonBlocking)添加payload

### addPayload方法

添加新的payload到state。如果是阻塞模式，会一直等到新的block被添加到payload buffer；否则，如果payload buffer满了会直接丢弃

### directMessage方法

如果是state请求消息，将消息转发到stateRequestCh；如果是state回应消息，将消息发送到stateResponseCh

### deliverPayloads方法

如果是s.payloads.Ready信号，将pop出的payload进行Unmarshal成一个Block对象，然后调用s.commitBlock方法提交区块。

如果是stopCh的信号，发送空对象到stopCh中，并返回

### commitBlock方法

调用s.committer.Commit提交区块。并调用g.gossip.UpdateChannelMetadata()更新通道信息

### autiEntropy方法

对于stopCh中的信号，将空对象发送到stopCh中，并返回

对于time.After(defAntiEntropyInterval)的信号，调用s.maxAvailableLedgerHeight()获得最高的账本高度，如果这个高度大于等于节点自身高度，调用s.requestBlocksInRange(uint64(current),uint(max))请求信区块

### maxAvailableLedgerHeight方法

调用s.gossip.PeerOfChannel获取可达节点信息，获取所有节点信息的最大账本高度

### requestBlocksInRange方法

每次请求defAutiEntropyBatchSize个区块。调用s.stateRequestMessage()创建state请求消息。调用s.selectPeerToRequestFrom方法获取请求区块的节点。然后调用s.gossip.Send(gossipMsg, peer)发送请求。

如果收到stateResponseCh中的信号，调用s.handleStateResponse方法，处理回应；如果超时，一直重试defAntiEntropyMaxRetries次；如果stopCh中的信号，发送空对象到stopCh并返回

### stateRequestMessage方法

创建一个proto.GossipMessage

```golang
return &proto.GossipMessage {
	Nonce: util.RandomUInt64(),
	Tag: proto.GossipMessage_CHAN_OR_ORG,
	Channel: []byte(s.chainID),
	Content: &proto.GossipMessage_StateRequest{
		StateRequest: &proto.RemoteStateRequest{
			StartSeqNum: benginSeq,
			EndSeqNum: endSeq,
		},
	},
}
```

### selectPeerToRequestFrom方法

调用s.filterPeers(s.hasRequiredHeight(height))获取拥有请求区块的节点列表；然后返回peers[util.RandomInt(n)]，即随机选择一个拥有请求的区块的节点。

### filterPeer方法

对于s.gossip.PeersOfChannel()的所有成员，判断其是否满足预设条件。

参数：

- predicate func(peer discovery.NetworkMember) bool

返回值：

- []*comm.RemotePeer

### hasRequiredHeight方法

返回是否持有给定高度的区块

```golang
return func(peer discovery.NetworkMember) bool {
	if nodeMetadata, err := FromBytes(peer.Metadata); err != nil {
		logger.Errorf("...")
	} else if nodeMetadata.LedgerHeight >= height {
		return true
	}
	return false
}
```

### handleStateResponse方法

从回应的消息中解析到payload的列表，针对每个payload调用s.addPayload(payload, blocking)将payload添加到payload buffer

### processStateRequests方法

对于s.stateRequestCh中的信号，调用s.handleStateRequest处理消息；对于stopCh中的信号，发送空对象到stopCh，并返回

### handleStateRequest方法

校验batchsize，读取当前leader状态获取区块，构建回应并发送出去。

参数：

- msg proto.ReceivedMessage

首先调用msg.GetGossipMessage().GetStateRequest()方法获取请求，校验request中的batchSize是否小于defAntiEntropyBatchSize。

构建回应

```golang
response := &proto.RemoteStateResponse{Payloads: make([]*[proto.Payload, 0])}
for seqNum := request.StartSeqNum; seqNum <= endSeqNum; seqNum++ {
	blocks := s.committre.GetBlocks([]uint64{seqNum})
	...
	blockBytes, err := pb.Marshal(blocks[0])
	...
	response.Payloads = append(reponse.Payloads, &proto.Payload{
		SeqNum: seqNum,
		Data: blockBytes,
	})
}
```

回送回应

```golang
msg.Response(&proto.GossipMessage {
	Nonce: msg.GetGossipMessage().Nonce,
	Tag: proto.GossipMessage_CHAN_OR_ORG,
	Channel: []byte(s.chainID),
	Content: &proto.GossipMessage_StateResponse{response},
})
```

### Stop方法

使用s.once.Do()保证执行一次

发送空对象到stopCh；调用s.done.Wait()保证所有的go-routine结束；调用s.committer.Close()关闭committer，关闭s.stateRequestCh，关闭s.stateResponseCh，关闭s.stopCh

### GetBlock方法

调用s.committer.GetBlocks()方法获取区块

### AddPayload方法

调用s.addPayload()方法添加payload到payload buffer


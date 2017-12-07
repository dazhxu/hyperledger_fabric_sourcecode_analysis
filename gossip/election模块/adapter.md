adapter
===

adapter文件定义了msgImpl类、peerImpl类、adapterImpl类及gossip接口

# gossip接口

定义了election模块用到的gossip基本功能

```golang
type gossip interface {
	Peers() []discovery.NetworkMember
	Accept(acceptor common.MessageAcceptor, passThrough bool) (<-chan *proto.GossipMessage, <- proto.ReceivedMessage)
	Gossip(msg *proto.GossipMessage)
}
```

# msgImpl对象

## 属性

包含*proto.GossipMessage属性

```golang
type msgImpl struct {
	msg *proto.GossipMessage
}
```

## 方法

### SenderID方法

获取发送者的PkiId

### IsProposal方法

是否是Proposal消息

### IsDeclaration方法

返回mi.msg.GetLeadershipMsg().IsDeclaration

# peerImpl对象

## 属性

包含discovery.NetworkMember属性

```golang
type peerImpl struct {
	member discovery.NetworkMember
}
```

## 方法

### ID方法

返回节点的PkiId

# adapterImpl对象

## 属性

包含gossip、selfPKIid/channel等对象

```golang
type adapterImpl struct{
	gossip gossip
	selfPKIid common.PKIidType
	incTime uint64
	seqNum uint64
	channel common.ChainID
	logger *logger.Logger
	doneCh chan struct{}
	stopOnce *sync.Once
}
```

## 构造方法

### NewAdapter方法

创建leader election adapter

属性：

- gossip gossip： gossip接口
- pkiid common.PKIidType
- channel common.ChainID

返回值：

- LeaderElectionAdapter

## 普通方法

### Gossip方法

调用ai.gossip.Gossip方法，发送消息

### Accept方法

调用ai.gossip.Accept()方法获取通道，只传送leadership org和channel信息

创建新线程，处理gossip消息

```golang
msgCh := make(chan Msg)

go func(inCh <-chan *proto.GossipMessage, outCh chan Msg, stopCh chan struct{}) {
	for {
		select {
		case <-stopCh:
			return
		case gossipMsg, ok := <-inCh:
			if ok {
				outCh <- &msgImpl{gossipMsg}
			} else {
				return
			}
		}
	}
}(adapterCh, msgCh, ai.doneCh)

return msgCh
```

### CreateMessage方法

创建leadership消息

```golang
ai.seqNum++
seqNum := ai.seqNum

leadershipMsg := &proto.LeadershipMessage{
	PkiId:         ai.selfPKIid,
	IsDeclaration: isDeclaration,
	Timestamp: &proto.PeerTime{
		IncNum: ai.incTime,
		SeqNum: seqNum,
	},
}

msg := &proto.GossipMessage{
	Nonce:   0,
	Tag:     proto.GossipMessage_CHAN_AND_ORG,
	Content: &proto.GossipMessage_LeadershipMsg{LeadershipMsg: leadershipMsg},
	Channel: ai.channel,
}

return &msgImpl{msg}
```

### Peers方法

返回ai.gossip.Peers()获取的节点

### Stop方法

关闭doneCh
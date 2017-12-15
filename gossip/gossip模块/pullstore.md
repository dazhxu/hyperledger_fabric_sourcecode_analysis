pullstore
===

# 常量及类型定义

## 常量

```golang
const (
	HelloMsgType MsgType = iota
	DigestMsgType
	RequestMsgType
	ResponseMsgType
)
```

## 类型定义

### MsgType类型

发送到PullStore的消息的类型

```golang
type MsgType int
```

### MessageHook函数类型

定义了在收到特定pull消息后应该运行的函数类型

```golang
type MessageHook func(itemIDs []string, items []*proto.SignedGossipMessage, msg proto.ReceivedMessage)
```

### DigestFilter函数类型

DigestFilter函数类型过滤发送到远端节点的digest

```golang
type DigestFilter func(helloMsg proto.ReceivedMessage) func(digestItem string) bool
```

DigestFilter函数类型拥有一个函数byContext，功能是将DigestFilter转换为algo.DigestFilter

```golang
func (df DigestFilter) byContext() algo.DigestFilter {
	return func(context interface{}) func(digestItem string) bool {
		return func(digestItem string) bool {
			return df(context.(proto.ReceivedMessage))(digestItem)
		}
	}
}
```

## 全局方法

### SelectEndpoints方法

从节点池中随机选择k个节点

# Sender接口

Sender接口可以发送消息到远端节点

```golang
type Sender interface {
	Send(msg *proto.SignedGossipMessage, peers ...*comm.RemotePeer)
}
```

# MembershipService接口

包含存活节点的成员信息

```golang
type MembershipService interface {
	GetMembership() []discovery.NetworkMember
}
```

# Config结构

定义了pull mediator的配置

```golang
type Config struct {
	ID 					string
	PullInterval 		time.Duration
	Channel 			common.ChainID
	PeerCountToSelect 	int
	Tag 				proto.GossipMessage_Tag
	MsgType 			proto.PullMsgType
}
```

# PullAdapter结构

PullAdapter定义了与gossip模块的不同结构进行交互的pullStore的方法

```golang
type PullAdapter struct {
	Sndr 		Sender
	MemSvc 		MembershipService
	IdExtractor proto.IdentifierExtractor
	MsgCons 	proto.MsgConsumer
	DigFilter 	DigesterFilter
}
```

# Mediator接口

Mediator接口包装了一个PullEngine，提供执行pull同步的方法。从pull mediator到一个特定类型的消息的具化有配置来完成，这个配置是一个IdentifierExtractor，在构造时传入，以及针对每个pull消息类型注册的hook。

```golang
type Mediator interface {
	Stop()
	RegisterMsgHook(MsgType, MessageHook)
	Add(*proto.SignedGossipMessage)
	Remove(digest string)
	HandleMessage(msg proto.ReceivedMessage)
}
```

- Stop方法：停止Mediator
- RegisterMsgHook方法：注册一个消息hook到特定的pull消息类型
- Add方法：添加一个gossip消息到Mediator
- Remove方法：从Mediator中移除与digest匹配的GossipMessage
- HandleMessage方法：处理来自远端节点的消息

# pullMediatorImpl类

pullMediatorImpl是对Mediator接口的实现

## 属性

pullMediatorImpl包含*PullAdapter，msgType2Hook，config，itemID2Msg，PullEngine等属性

```golang
type pullMediatorImpl struct {
	sync.RWMutex
	*PullAdapter
	msgType2Hook 	map[MsgType][]MessageHook
	config 			Config
	logger 			*logging.Logger
	itemID2Msg 		map[string]*proto.SignedGossipMessage
	engine 			*algo.PulEngine
}
```

## 构造方法

### NewPullMediator方法

创建一个新的Mediator对象

参数：

- config Config
- adapter *PullAdapter

返回值：

- Mediator

首先获取adapter的DigFilter，如果不存在创建一个接受所有消息的Filter。

创建pullMediatorImpl，并设置pullMediator的engine属性

```golang
p := &pullMediatorImpl {
	PullAdapter: adapter,
	msgType2Hook: make(map[MsgType][]MessageHook),
	config: config,
	logger: util.GetLogger(util.LoggongPullModule, config.ID),
	itemID2Msg: make(map[string]*proto.SignedGossipMessage),
}
p.engine = algo.NewPullEngineWithFilter(p, config.PullInterval, digFilter.byContext())
return p
```

## 普通方法

### HandleMessage方法

处理远端节点发来的各种消息

参数：

- m proto.ReceivedMessage

首先，获取m的message数据和message类型

```golang
if m.GetGossipMessage == nil || !m.GetGossipMessage().IsPullMsg() {
	return
}
msg := m.GetGossipMessage()
msgType := msg.GetPullMsgType()
if msgType != p.config.MsgType {
	return
}
```

根据msg的类型，对msg进行处理。如果是hello消息，调用p.engine.OnHello方法；如果是digest消息，调用p.engine.OnDigest方法；如果是Request方法，调用p.engine.Onreq方法；如果是response消息，调用p.engine.OnRes方法；

```golang
if helloMsg := msgGetHello(); helloMsg != nil {
	pullMsgType = HelloMsgType
	p.engine.OnHello(helloMsg.Nonce, m)
}

if digest := GetDataDig(); digest != nil {
	itemIDs = digest.Digests
	pullMsgType = DigestMsgType
	p.engine.OnDigest(digest.Digests, digest.Nonce, m)
}

if req := msg.GetDataReq(); req != nil {
	itemIDs = req.Digests
	pullMsgType = RequestMsgType
	p.engine.OnReq(req.Digest, req.Nonce, m)
}

if res := msg.GetDataUpdate(); res != nil {
	itemIDs = make([]string, len(res.Data))
	items = make([]*proto.SignedGossipMessage, len(res.Data))
	pullMsgType = ResponseMsgType
	for i, pulledMsg := range res.Data {
		msg, err := pulledMsg.ToGossipMessage()
		...
		p.MsgCons(msg)
		itemIDs[i] = p.IdExtractor(msg)
		items[i] = msg
		p.Lock()
		p.itemID2Message[itemIDs[i]] = msg
		p.Unlock()
	}
	p.engine.OnRes(itemIDs, res.Nonce)
}
```

最后调用hook方法处理items

```golang
for _, h := range p.hooksByMsgType(pyllMsgType) {
	h(itemIDs, items, m)
}
```

### Stop方法

停止pullMediatorImpl实例。调用p.engine.Stop()

### RegisterMsgHook方法

将消息hook注册到特定的pull消息类型上

参数：

- pullMsgType MsgType
- hook MessageHook

```golang
p.Lock()
defer p.Unlock()
p.msgType2Hook[pullsgType] = append(p.msgType2Hook[pullMsgType], hook)
```

### Add方法

添加一个GossipMessage到store

```golang
itemID := p.IdExtractor(msg)
p.itemID2Msg[itemID] = msg
p.engine.Add(itemID)
```

### Remove方法

将digest消息从Mediator中移除

```golang
delete(p.itemID2Msg, digest)
p.engine.Remove(digest)
```

### SelectPeers方法

选择p.config.PeerCountToSelect个节点

### Hello方法

发送hello消息初始化pull协议

参数：

- dest string
- nonce uint64

首先创建一个GossopMessage

```golang
helloMsg := &proto.GossipMessage {
	Channel: p.config.Channel,
	Tag: p.config.Tag,
	Content: &proto.GossipMessage_Hello{
		Hello: &proto.GossipHello{
			Nonce: nonce,
			Metadata: nil,
			MsgType: p.config.MsgType,
		}
	}
}
```

不对消息签名，调用p.Sndr.Send方法将消息发送出去

```golang
sMsg, err := helloMsg.NoopSign()
p.Sndr.Send(sMsg, p.peersWithEndpoint(dest))
```

### SendDigest方法

发送digest到远端节点

参数：

- digest []string
- nonce uint64
- context interface{}

首先，创建一个GossipMessage

```golang
digMsg := &proto.GosspMessage{
	Channel: p.config.Channel,
	Tag: p.config.Tag,
	Nonce: 0,
	Content: &proto.GossipMessage_DataDig{
		DataDig: &proto.DataDigest{
			MsgType: p.config.MsgType,
			Nonce: nonce,
			Digests: digest,
		}
	}
}
```

回送消息

```golang
context.(proto.ReceivedMessage).Respond(digMsg)
```

### SendReq方法

发送一组item到远端的PullEngine实例

参数：

- dest string
- item []string
- nonce uint64

首先创建一个GossipMessage

```golang
req := &proto.GossipMessage {
	Channel: p.config.Channel,
	Tag: p.config.Tag,
	Nonce: 0,
	Content: &proto.GossipMessage_DataReq{
		DataReq: &proto.DataRequest{
			MsgType: p.config.MsgType,
			Nonce: nonce,
			Digests: items,
		},
	},
}
```

不对消息签名，并调用p.Sndr.Send()方法发送出去

```golang
sMsg, err := req.NoopSign()
p.Sndr.Send(sMsg, p.peersWithEndpoint(dest)...)
```

### SendRes方法

发送一组item到特定的远端PullEngine实例

参数：

- items []string
- context interface{}

首先，从p.itemID2Msg[item]中获取要返回的msg

然后创建一个GossipMessage

```golang
returedUpdate := &proto.GossipMessage{
	Channel: p.config.Channel,
	Tag: p.config.Tag,
	Nonce: 0,
	Content: &proto.GossipMessage_DataUpdate{
		DataUpdate: &proto.DataUpdate{
			MsgType: p.config.MsgType,
			Nonce: nonce,
			Digests: items2return,
		},
	},
}
```

然后，调用context.(proto.ReceivedMessage).Response(returnedUpdata)将消息回送给发送请求的远端节点。

### peersWithEndpoints方法

返回endpoint对于的peer

参数：

- endpoints ...string

返回值：

- []*comm.RemotePeer

### hooksByMsgType方法

根据消息类型返回处理函数

参数：

- msgType MsgType

返回值：

- []MessageHook
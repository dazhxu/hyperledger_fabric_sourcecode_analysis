discovery
===

# 方法类型定义

## EnvelopeFilter类型

EnvelopeFiltr方法可能会从产生SignedGossipMessage的Envelope中移除部分数据

```golang
type EnvelopeFilter func(message *proto.SignedGossipMessage) *proto.Envelope
```

## Sieve类型

Sieve类型的方法定义了基于某些标准消息是否允许被发送到远端节点。

```golang
type Sieve func(message *proto.SignedGossipMessage) bool
```

## DisclosurePolicy类型

DisclosurePolicy类型定义了某个远端节点应该可以感知的那些消息，以及在给定的SignedGossipMessage外哪些是可以被知道的。

```golang
type DisclosurePolicy func(remotePeer *NetworkMember) (Sieve, EnvelopeFilter)
```

# CryptoServer接口

discovery模块期望实现的加密相关的接口

## ValidateAliveMsg方法

验证Alive消息是否有效

参数：

- message *proto.SignedGossipMessage

返回值：

- bool

## SignedMessage方法

签名一个消息

参数：

- m *proto.GossipMessage
- internalEndpoint string

返回值：

- *proto.Envelope

## CommService接口

CommService是discovery接口期望被实现的并在创建时被传递的接口

### Gossip方法

gossip一个消息

参数：
 
- msg *proto.SignedGossipMessage

### SendToPeer方法

向某个特定的节点发送消息

参数：

- peer *NetworkMember
- msg *proto.SignedGossipMessage

### Ping方法

探测一个节点，返回其是否有回应

参数：

- peer *NetworkMember

返回值：

- bool

### Acceptor方法

返回一个只读的通道，用来传输远端节点发送的程membership消息

返回值：

- <- chan *proto.SignedGossipMessage

### PresumedDead方法

返回一个只读的通道，用来传递被认为是不通的节点

返回值：

- <-chan common.PKIidType

### CloseConn方法

关闭与某个peer的连接

参数：

- peer *NetworkMember

## NetworkMember方法

代表了peer

```golang
type NetworkMember struct {
	Endpoint 			string
	Metadata 			[]byte
	PKIid 				common.PKIidType
	InternalEndpoint 	string
}
```

## Discovery接口

代表了Discovery模块

### Lookup方法

返回networkMember，如果没找到，返回nil

```golang
Lookup(PKIID common.PKIidType) *NetworkMember
```
### Self方法

返回自己的membership信息

```golang
Self() NetworkMember
```

### UpdateMetadata方法

更新这个实例的metadata

```golang
UpdateMetadata([]byte)
```

### UpdateEndpoint方法

更新这个实例的endpoint

```golang
UpdateEndpoint(string)
```

### Stop方法

停止这个实例

### GetMembership方法

返回连接的成员

```golang
GetMembership() []NetworkMember
```

### InitiateSync方法

使这个实例询问给定数目的peer，他们的成员信息

```golang
InitiateSync(peerNum int)
```

### Connect方法

使此实例连接到远端节点

```golang
Connect(member NetworkMember, id identifier)
```

参数identifier用来识别peer，判断是否在peer的组织，以及动作是否成功

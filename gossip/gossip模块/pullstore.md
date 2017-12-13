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

# 全局方法

## SelectEndpoints方法

从节点池中随机选择k个节点

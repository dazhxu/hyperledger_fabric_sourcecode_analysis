msg
===

msg文件定义了ReceivedMessageImpl类

# ReceivedMessageImpl类

## 属性

ReceivedMessageImpl继承了proto.SignedGossipMessage，包含了连接和链接信息等属性

```golang
type ReceivedMessageImpl struct {
	*proto.SignedGossipMessage
	lock sync.Locker
	conn *connection
	connInfo *proto.ConnectionInfo
}
```

## 方法

### GetSourceEnvelope方法

返回m.Envelope

### Response方法

发送一个消息到ReceivedMessageImpl的来源

### GetGossipMessage方法

返回m.SignedGossipMessage

### GetConnectionInfo方法

返回m.connInfo
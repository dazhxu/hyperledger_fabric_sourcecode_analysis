common
===

commom模块定义了一些基本的数据结构和数据类型

# 基本数据类型

## PKIidType类型

PKI-id代表了节点的安全的身份

```golang
type PKIidType byte[]
```

## ChainID类型

代表chain的标识

```golang
type ChainID []byte
```

## MessageAcceptor函数类型

此函数用来觉得创建MessageAcceptor的订阅者对那些消息感兴趣

```golang
type MessageAcceptor func(interface{}) bool
```

## MessageReplacingPolicy函数类型

返回InvalidationResult，即在放入gossip消息存储时，一个消息和另一个消息的关系。

```golang
type MessageReplacingPolicy func(this interface{}, that interface{}) InvalidationResult
```

- 如果this验证that，则返回MESSAGE_INVALIDATES；
- 如果this被that验证，则返回MESSAGE_INVALIDATED;
- 否则，返回MESSAGE_NO_ACTION

# 基本数据结构

## Payload对象

common模块定义了Payload对象，此对象包含一个账本区块

```golang
type Payload struct {
	ChainID ChainID
	Data 	[]byte
	Hash	string
	SeqNum	uint64
}
```
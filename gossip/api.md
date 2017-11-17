api
===

api模块定义了channel相关的数据结构和接口，以及crypto相关的数据结构和接口。

# channel文件

## SecurityAdvisor接口

SecurityAdvisor接口提供安全和身份相关的辅助功能，包含一个方法OrgByPeerIdentity，给定peer的identity返回OrgIdentityType。此方法不验证peer身份。

## ChannelNotifier接口

由gossip模块实现，被peer层使用，向gossip模块通知JoinChannel事件。包含JoinChannel方法。

```golang
JoinChannel(joinMsg JoinChannelMessage, chainID common.ChainID)
```

## JoinChannelMessage接口

JoinChannelMessage是维护channel成员列表生成和变化的消息，在peer之间进行gossip传播。包含三个方法：SequenceNumber、Members、AnchorPeersOf。

- SequenceNumber方法

返回JoinChannelMessage来自的配置的Sequence Number

- Members方法

返回通道中的组织

- AnchorPeerOf方法

返回给定OrgIdentity对应的Anchor节点。

## AnchorPeer结构

```golang
type AnchorPeer struct {
	Host string
	Port int
}
```

## OrgIdentityType类型

定义组织的身份的类型

```golang
type OrgIdentityType []byte
```

# crypto文件

crypto文件定义了MessageCryptoService接口和其他的数据类型

## PeerIdentityType

定义了peer证书的类型

```golang
type PeerIdentityType []byte
```

## PeerSuspector函数类型

判断一个给定identity的peer，其身份或CA是否被吊销。

## PeerSecureDialOpts函数类型

返回一个与远端peer通信时使用通信级安全的gRpc连接选项。

```golang
PeerSecureDialOpts func() []grpc.DialOption
```

## MessageCryptoService接口

MessageCryptoService接口是gossip模块与peer的安全层之间的约定。Gossip模块用它来校验和认证远程节点及它发送的数据，以及验证从ordering服务接收到的区块。

- GetPKIidOfCert方法

返回给定身份的节点的pkiID

- VerifyBlock方法

如果区块被正确签名，并且seqNum包含在区块头中，则返回nil；否则返回错误。

- Sign方法

用peer的签名密钥对消息进行签名

- Verify方法

验证消息是否是由给定peer签名证书签名的。如果验证通过，返回nil；否则返回错误。

- VerifyByChannel方法

验证消息是否是由给定peer的签名证书在特定的通道上下文下签名的

- ValidateIdentity方法

校验远端peer的身份。如果身份是无效的、吊销的、过期的，则返回错误；否则返回nil。
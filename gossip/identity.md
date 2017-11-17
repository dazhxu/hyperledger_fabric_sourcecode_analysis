identity
===

identity模块定义了管理节点身份的接口和对象

# storedIdentity对象

storedIdentity对象存储了peerIdentity以及其操作

## storedIdentity属性

包括最近访问的时间和peerIdentity

```golang
type storedIdentity struct {
	lastAccessTime int64
	peerIdentity api.PeerIdentityType
}
```

## storedIdentity方法

storedIdentity的方法有fetchIdentity方法和fetchLastAccessTime方法

# Mapper接口与对象

## Mapper接口

Mapper接口维护了节点pki-id到节点证书的映射，接口用于以下方法

- Put方法

将节点的identity与pkiID关联起来

- Get方法

获取给定pkiID的identity

- Sign方法

签名一个消息，如果成功返回签名后的消息，失败返回错误

- Verify方法

验证一个签名的消息

- GetPKIidOfCert

获取给定cert的pkiID

- ListInvalidIdentities方法

返回吊销、过期或长时间不用的节点证书对应的pkiID列表

```golang
ListInvalidIdentities(isSuspected api.PeerSuspector) []common.PKIidType
```

## identityMapperImpl对象

identityMapperImpl对象是Mapper接口的一个实现。

```golang
type identityMapperImpl struct {
	mcs			api.MessageCryptoService
	pkiID2Cert	map[string]*storedIdentity
	sync.RWMutex
	selfPKIID string
}
```

### 构造函数

构造函数需要传入一个api.MessageCryptoService的引用和selfIdentity

### identityMapperImpl的方法

- Put方法

将pkiID与identity连接起来

- Get方法

获取pkiID所对应的identity

- Sign方法

调用mcs对象的Sign方法对消息进行签名

- Verify方法

调用mcs对象的Verify方法对消息进行签名

- GetPKIidOfCert方法

调用mcs对象的GetPKIidOfCert方法获取identity对应的PKI-ID


- validateIdentities方法

遍历pkiID2Cert，调用isSuspected函数判断identity对象是否为吊销、过期或长时间未用。返回suspected节点identity的列表

- ListInvalidIdentities方法

列出所有suspected节点id，并从pkiID2Cert中移除


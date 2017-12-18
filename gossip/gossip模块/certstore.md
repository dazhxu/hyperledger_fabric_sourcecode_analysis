certstore
===

certstore模块中定义了certStore类，用于支持身份消息的pull分发

# certStore结构

## 属性

```golang
type certStore struct {
	sync.RWMutex
	selfIdentity 	api.PeerIdentityType
	idMapper 		identity.Mapper
	pull 			pull.Mediator
	logger 			*logging.Logger
	mcs 			api.MessageCryptoService
}
```

## 构造方法

### newCertStore方法

创建一个certStore对象

参数：

- puller pull.Mediator
- idMapper identity.Mapper
- selfIdentity api.PeerIdentityType
- mcs api.MessageCryptoService

返回值：

- *certStore

首先创建certStore对象。然后将selfPKIID和selfIdentity放入certStore.idMapper.Put方法.

调用certStore.createIdentityMessage()创建identity消息，并调用puller.Add(selfIdMsg)将消息添加到pull.Meidator。注册MsgHook。最后返回certStore对象

```golang
puller.RegisterMsgHook(pull.RequestMsgType, func(_ []string, msgs []*proto.SignedGossipMessage, _ proto.ReceivedMessage)) {
	for _, msg := range msgs {
		pkiID := common.PKIidType(msg.GetPeerIdentity().PkiId)
		cert := api.PeerIdentityType(msg.GetPeerIdentity().Cert)
		if err := certStore.idMapper.Put(pkiID, cert); err != nil {
			...
		}
	}
}
```

### handleMessage方法

处理消息

```golang
if update := msg.GetGossipMessage().GetDataUpdate();update!=nil {
	for _, env := range update.Data {
		m, err := env.ToGossipMessage()
		...
		if !m.IsIdentityMsg() {
			return
		}
		if err := cs.validateIdentityMsg(m);err!=nil {
			return
		}
	}
}
cs.pull.HandleMessage(msg) 
```

### validateIdentityMsg方法

校验消息

首先从消息中获取pkiid和cert等。

```golang
idMsg := msg.GetPeerIdentity()
...
pkiID := idMsg.PkiId
cert := idMsg.Cert
calculatedPKIID := cs.mcs.GetPKIidOfCert(api.PeerIdentityType(cert))
claimedPKIID := common.PKIidType(pkiID)
if !bytes.Equal(calculatedPKIID, claimedPKIID) {
	return fmt.Errorf(...)
}
```

然后创建验证器，验证消息，最后调用cs.mcs.ValidateIdentity方法验证身份

```golang
verifier := func(peerIdentity []byte, signature, message []byte) error {
	return cs.mcs.Verify(api.PeerIdentityType(peerIdentity), signature, message)
}
err := msg.Verify(cert, verifier)
...
return cs.mcs.ValidateIdentity(api.PeerIdentity(idMsg.Cert))
```

### createIdentityMessage方法

创建Identity消息

首先，创建proto.PeerIdentity，然后根据创建proto.GossipMassage，最后用cs.idMapper对消息签名。

```golang
identity := &proto.PeerIdentity{
	Cert: 		cs.selfIdentity,
	Metadata: 	nil,
	PkiId: 		cs.idMapper.GetPKIidOfCert(cs.selfIdentity),
}
m := &proto.GossipMessage{
	Channel: il,
	Nonce: 0,
	Tag: proto.GossipMessage_EMPTY,
	Content: &proto.GossipMessage_PeerIdentity{
		PeerIdentity: identity,
	}
}
signer := func(msg []byte) ([]byte, error) {
	return cs.idMapper.Sign(msg)
}
sMsg := &proto.SignedGossipMessage{
	GossipMessage: m,
}
_, err := sMsg.Sign(signer)
return sMsg, err
```

### listRevokedPeers方法

返回失效的节点

参数：

- isSuspected api.PeerSuspector

返回值：

- []common.PKIidType

调用cs.idMapper.ListIncalidIdentities(isSuspected)获取revokedPeers，针对revokedPeers的每个pkiID，调用cs.pull.Remove(string(pkiID))从pullMediator中移除，返回revokedPeers

### stop方法

调用cs.pull.Stop停止模块
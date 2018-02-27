configupdate
===

configupdate是broadcast.Processor接口的实现。它促进了CONFIG_UPDATE的预处理。

# SupportManager接口

SupportManager接口提供了为chain查找Support的方式

```golang
type SupportManager interface {
	GetChain(chainID string) (Support, bool)
	NewChannelConfig(envConfigUpdate *cb.Envelope) (configtxapi.Manager, error)
}
```

# Support接口

Support接口列举了全通道support函数的子集。

```golang
type Support interface {
	ProposeConfigUpdate(env *cb.Envelope) (*cb.ConfigEnvelope, error)
}
```

# Processor结构

## 属性

包括一个签名器、一个SupportManager及系统通道ID和系统通道support

```golang
type Processor struct {
	signer 					crypto.LocalSigner
	manager 				SupportManager
	systemChannelID			string
	systemChannelSupport 	Support
}
```

## 构造方法

### New方法

首先调用support.Manager的GetChain方法获取support实例，然后构造一个Processor实例

```golang
support, ok := supportManager.GetChain(systemChannelID)
...
return &Processor{
	systemChannelID: 		systemChannelID,
	manager: 				supportMananger,
	signer: 				signer,
	systemChannelSupport:	support,
}
```

## 普通方法

### Process方法

Process方法接受CONFIG_UPDATE类型的包，将其转换为一个创建新通道请求，或者是通道更新交易

首先，调用channelID(envConfigUpdate)获取channelID；然后调用p.manager.GetChain(channelID)获取support对象；如果support存在，调用p.existingChannelConfig处理通道再配置请求；否则调用p.newChannelConfig处理创建通道请求。

### channelID方法

从cb.Envelope中获取channelID 

```golang
envPayload, err := utils.UnmarshalPayload(env.Payload)
...
chdr, err := utils.UnmarshalChannelHeader(envPayload.Header.ChannelHeader)
...
return chdr.ChannelId, nil
```

### existingChannelConfig方法

调用support.ProposeConfigUpdate(envConfigUpdate)获取configEnvelope，然后调用utils.CreateSignedEnvelope(cb.HeaderType_CONFIG, channelID, p.siger, configEnvelope, msgVersion, epoch)生成签名的envelope

### newChannelConfig方法

首先调用p.manager.NewChannelConfig(envConfigUpdate)方法，然后调用ctxm.ProposeConfigUpdata方法创建configEnvelope，然后调用utils.CreateSignedEnvelope方法进行签名，最后调用p.proposeNewChannelToSystemChannel方法。

### proposeNewChanneToSystemChannel方法

调用utils.CreateSignedEnvelope(cb.HeaderType_ORDERER_TRANSACTION, p.systemChannelID, p.signer, newChannelEnvConfig, msgVersion, epoch)
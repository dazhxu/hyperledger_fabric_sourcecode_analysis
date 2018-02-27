systemchain
===

定义与系统通道相关的接口与结构

# chainCreator接口

为更简单的模拟定义了一些内部接口

```golang
type chainCreator interface {
	NewChannelConfig(envConfigUpdate *cb.Envelope) (configtxapi.Manager, error)
	newChain(configTx *cb.Envelope)
	channelsCount() int
}
```

# limitedSupport接口

定义共享配置的接口

```golang
type limitedSupport interface {
	SharedConfig() config.Orderer
}
```

# systemChainCommitter结构

## 属性

包含systemChainFilter和cb.envelope属性

## 方法

### Isolated方法

返回true

## Commit方法

调用filter.cc.newChain(scc.configTx)方法创建chain

# systemChainFilter结构

## 属性

包括chainCreator和limitedSupport属性

## 构造方法

### newSystemChainFilter方法

创建一个systemChainFilter实例

## 普通方法

### Apply方法

接受一个cb.Envelope，返回filter.Action和filter.Committer

首先将env.Payload数据Unmarshal成cb.Payload。然后调用utils.UnmarshalChannelHeader。将msgData.Data数据Unmarshal成cb.Envelope，然后调用authorizeAndInspect方法进行验证。最后返回filter.Accep和&systemChainCommitter。

### authorizeAndInspect方法

校验cb.Envelope。首先将cb.Envelope转换成cb.Payload，然后调用utils.UnmarshalChannelHeader,校验是不是cb.HeaderType_CONFIG。然后调用authorize方法进行验证，最后调用inspect方法判断config是不是修改了任何的orderer。

### authorize方法

首先调用scf.cc.NewChannelConfig方法根据configEnvelope.LastUpdate创建channelconfig。调用configManager.ProposeConfigUpdate方法创建newChannelConfigEnv，调用configManager.Apply方法，返回configMananger

### inspect方法

判断proposeManager与configManager的ConfigEnvelope是否相同。
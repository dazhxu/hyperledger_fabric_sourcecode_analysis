eventer
===

eventer模块定义了config时间相关的结构

# Config接口

Config接口列举了gossip需要的配置方法

```golang
type Config interface {
	ChainID() string
	Organizations() map[string]config.ApplicationOrg
	Sequence() uint64
}
```

# ConfigProcessor接口

接受config更新

```golang
type ConfigProcessor interface {
	ProcessConfigUpdate(config Config)
}
```

# configEventReceiver接口

包含configUpdated方法

# configStore结构

configStore保存了anchorpeer和org

```golang
type configStore struct {
	anchorPeers []*peer.AnchorPeer
	orgMap 		map[string]config.ApplicationOrg
}
```

# appGrp结构

## 属性

包括name, msgID, anchorPeers属性

```golang
type appGrp struct {
	name string
	mspID string
	anchorPeers []*peer.AnchorPeer
}
```

## 方法

### Name方法

返回ag.name

### MSPID方法

return ag.msgID

### AnchorPeers方法

返回ag.anchorPeers

# configEventer结构

## 属性

包含一个configStore和一个configEventReceiver

```golang
type configEventer struct {
	lastConfig *configStore
	receiver   configEventReceiver
}
```

## 构造方法

### newConfigEventer方法

创建一个新的configEnventer

## 普通方法

### ProcessConfigUpdate方法

无论是channel配置在初始化时或者更新时，都应该调用此方法。

首先调用cloneOrgConfig复制组织。创建新的configStore实例，最后调用ce.receiver.configUpdated(config)

### cloneOrgConfig方法

```golang
clone := make(map[string]config.ApplicationOrg)
	for k, v := range src {
		clone[k] = &appGrp{
			name:        v.Name(),
			mspID:       v.MSPID(),
			anchorPeers: v.AnchorPeers(),
		}
	}
return clone
```
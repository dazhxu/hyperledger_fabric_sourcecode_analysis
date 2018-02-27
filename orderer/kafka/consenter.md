consenter
===

# closeable接口

关闭调用资源的接口

```golang
type closeable interface {
	close() error
}
```

# commonConsenter接口

commonConsenter可以从consenter对象中获取配置选项，由consenterImpl实现

```golang
type commonConsenter interface {
	brokerConfig() *sarama.Config
	retryOptions() localconfig.Retry
}
```

# consenterImpl

consenterImpl实现了multichain.Consenter接口

## 属性

consenterImpl类的属性由各种配置组成

```golang
type consenterImpl struct {
	brokerConfigVal *sarama.Config
	tlsConfigVal 	localconfig.TLS
	retryOptionsVal localconfig.Retry
	kafkaVersionVal sarama.KafkaVersion
}
```

## 构造方法

创建基于kafka的共识器。由orderer模块的main.go调用

参数：

- tlsConfig localconfig.TLS
- retryOptions localconfig.Retry
- kafkaVersion sarama.KafkaVersion

返回值：

- multichain.Consenter

首先根据参数创建brokerConfig对象，然后返回一个consenterImpl

```golang
brockerConfig := newBrokerConfig(tlsConfig, retryOptions, kafkaVersion, defaultPartition)
return &consenterImpl {
	brokerConfigVal: brokerConfig,
	tlsConfigVal: tlsConfig,
	retryOptionsVal: retryOptions,
	kafkaVersionVal: kafkaVersion
}
```

## 普通方法

### HandleChain方法

HandleChain创建并返回一个指向multichain.Chain对象的引用，实现multichain.Consenter接口。被multichain.newChainSupport()调用，而后者被multichain.NewManagerImpl()调用

参数：

- support multichain.ConsenterSupport
- metadata *cb.Metadata

返回值：

- multichain.Chain
- error

```golang
lastOffsetPersisted := getLastPersisted(metadata.Value, support.ChainID())
return newChain(consenter, support, lastOffsetPersisted)
```

### brokerConfig方法

返回brokerConfigVal属性

### retryOptions方法

返回retryOptionsVal属性
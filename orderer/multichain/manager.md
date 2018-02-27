manager
===

# Manager接口

Manager处理chain的创建和访问

```golang
type Manager interface {
	GetChain(chainID string) (ChainSupport, bool)
	SystemChannelID() string
	NewChannelConfig(envConfigUpdate *cb.Envelope) (configtxapi.Manager, error)
}
```

# configResources结构

## 属性

包含configtxapi.Manager属性

## 方法

### SharedConfig方法

返回一个config.Orderer

调用cr.OrdererConfig方法创建config.Orderer

# ledgerResources结构

包含*configResources和ledger.ReadWrite属性

# multiLedger结构

## 属性

包含string到chainSupport和Consenter的映射，及ledgerFactory、signer及系统channel

```golang
type multiLedger struct {
	chains 			map[string]*chainSupport
	consenters 		map[string]*Consenter
	ledgerFactory	ledger.Factory
	signer 			crypto.LocalSigner

}
```

## 构造方法

### NewManagerImpl方法

创建一个Manger实例

参数：

- ledgerFactory ledger.Factory
- consenters map[string]Consenter
- signer crypto.LocalSigner

返回值：

- Manager

从ledgerFactory中获取chainID。对于每个
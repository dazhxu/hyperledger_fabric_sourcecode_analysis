chainsupport
===

# Consenter接口

Consenter接口定义了排序机制

```golang
type Consenter interface {
	HandleChain(support ConsenterSupport, metadata *cb.Metadata) (Chain, error)
}
```

# Chain接口

Chain定义了插入排序消息的方式

```golang
type Chain interface {
	Enqueue(env *cb.Envelope) bool
	Errored() <-chan struct{}
	Start()
	Halt()
}
```

# ConsenterSupport接口

提供实现一个Consenter所需要的资源

```golang
type ConsenterSupport interface {
	crypto.LocalSigner
	BlockCutter() blockcutter.Receiver
	SharedConfig() config.Orderer
	CreateNextBlock(messages []*cb.Envelope) *cb.Block
	WriteBlock(block *cb.Block, committers []filter.Committer, encodedMetadataValue []byte) *cb.Block
	ChainID() string
	Height() uint64
}
```

# ChainSupport接口

ChainSupport提供了支持chain的资源的包装

```golang
type ChainSupport interface {
	PolicyManager() policies.Manager
	Reader() ledger.Reader
	Errored() <-chan struct{}
	broadcast.Support
	ConsenterSupport
	Sequence() uint64
	ProposeConfigUpdate(env *cb.Envelope) (*cb.ConfigEnvelope, error)
}
```

# chainSupport结构

ChainSupport接口的实现

## 属性

chainSupport包含ledgerResouces, Chain, blockcutter.Receiver, filter.RuleSet等属性

```golang
type chainSupport struct {
	*ledgerResources
	chain Chain
	cutter blockcutter.Receiver
	filters *filter.RuleSet
	Signer crypto.LocalSigner
	lastConfig uint64
	lastConfigSeq uint64
}
```

## 构造方法

### newChainSupport方法

创建一个chainSupport对象

参数：

- filter *filter.RuleSet
- ledgerResources *ledgerResources
- consenters map[string]Consenter
- signer crypto.LocalSigner

返回值：

- *chainSupport

首先调用blockcutter.NewReceiverImpl方法创建blockcutter;然后调用ledgerResources.SharedConfig().ConsensusType()方法获取共识类型，并构建consenter。

构造chainSupport对象。调用cs.Sequence()方法创建last.ConfigSeq。然后调用ledger.GetBlock()方法获取最近的区块。如果最近的区块不是0，那么调用utils.GetLastConfigIndexFromBlock方法初始化lastConfig。然后调用utils.GetMetadataFromBlock()方法获取metadata。用cs和metadata调用consenter.HandleChain()方法创建cs.chain。

最后返回cs对象

## 普通方法

### createStandardFilters方法

为非系统chain创建filters的集合

```golang
return filter.NewRuleSet([]filter.Rule{
	filter.EmptyRejectRule,
	sizefilter.MaxBytesRule(ledgerResources.SharedConfig()),
	sigfilter.New(policies.ChannelWriters, ledgerResources.PolicyManager()),
	configtxfilter.NewFilter(ledgerResourcs),
	filter.AcceptRule,
})
```

### createSystemChainFilters方法

为ordering系统chain创建filters集合

### start方法

调用cs.chain.Start方法，启动链

### NewSignatureHeader方法

调用cs.signer.NewSignatureHeader方法创建签名头

### Sign方法

调用cs.signer.Sign(message)方法签名消息

### Filters方法

返回cs.filters

### BlockCutter方法

返回cs.cutter

### Reader方法

返回cs.ledger

### Enqueue方法

调用cs.chain.Enqueue方法

### Errored方法

调用cs.chain.Errored方法

### CreateNextBlock方法

调用ledger.CreateNextBlock创建下一个区块

### addBlockSignature方法

添加区块签名

参数：

- block *cb.Block

首先获取blockSignature

```golang
blockSignature := &cb.MetadataSignature{
	SignatureHeader: utils.MarshalOrPanic(utils.NewSignatureHeaderOrPanic(cs.signer))
}
```

调用utils.SignOrPanic进行签名，并将签名写入区块中

```golang
blockSignature.Signature = utils.SignOrPanic(cs.signer, util.ConcatenateBytes(blockSignatureValue, blockSignature.SignatureHeader, block.Header.Bytes()))

block.Metadata.Metadata[cb.BlockMetadataIndex_SIGNATURES] = utils.MarshalOrPanic(&cb.Metadata{
	Value: blockSignatureValue,
	Signatures: []*cb.MetadataSignature{
		blockSignature,
	},
})
```

### addLastConfigSignature方法

对cs.lastConfigSignature进行签名并写入区块中

### WriteBlock方法

将区块写入账本

参数：

- block *cb.Block
- committers []filter.Committer
- encodedMetadataValue []byte

返回值：

- *cb.Block

首先，对committers中的每个committer执行提交操作；然后设置与orderer相关的metadata区域；调用addBlockSignature添加区块的签名；调用addLastConfigSignature方法添加最近配置签名；调用cs.ledger.Append(block)方法将区块添加到账本中

### Height方法

调用cs.Reader().Height()方法返回账本高度。
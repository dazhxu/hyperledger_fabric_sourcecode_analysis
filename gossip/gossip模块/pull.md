pull
===

pull文件定义了PullEngine, PullEngine是运行基于pull的gossip，并维护用数字串标识的内部状态。协议的流程：

1) 初始化器向一组远端节点发送带特定Nonce的Hello消息
2) 每个远端节点回送它的消息的摘要，及Nonce
3) 初始化器检查Nonce的有效性，并收集消息摘要，构建包含特定ids的请求，并将每个请求发送到对应的peer
4) 每个Peer回应所请求的items，及nonce

# 常量及方法定义

## 常量

```golang
const (
	defDigestWaitTime = time.Duration(1) * time.Second
	defRequestWaitTime = time.Duration(1) * time.Second
	defResponseWaitTime = time.Duration(2) * time.Second
)
```

## 全局方法

### SetDigestWaitTime方法

设置digest等待时间

### SetRequestWaitTime方法

设置请求等待时间

### SetResponseWaitTime方法

设置回应等待时间

## DigestFilter函数类型

DigestFilter根据消息的context过滤发送hello和request消息的远端节点发送过来的digest

# PullAdapter接口

PullEngine需要PullAdapter来发送消息到远端的PullEngine实例。当从远端PullEngine实例收到相应的消息，将会调用PullEngine接口的OnHello、OnDigest、OnReq和OnRes方法

```golang
type PullAdapter interface {
	SelectPeers() []string
	Hello(dest string, nonce uint64)
	SendDigest(digest []string, nonce uint64, context interface{})
	SendReq(dest string, items []string, nonce uint64)
	SendRes(items []string, context interface{}, nonce uint64)
}
```

- SelectPeers方法：选择一组节点，engine与之初始化协议
- Hello方法：发送hello消息来初始化协议，nonce期望在digest消息中被返回
- SendDigest方法：发送一个digest消息到远端PullEngin实例。
- SendReq方法：发送一组条目到特定的某个PullEngine
- SendRes方法：发送一组条目到被context标识的远端PullEngin实例

# PullEngine类

PullEngine是在PullAdapter的帮助下执行pull算法的组件

## 属性

PullEngine继承PullAdapter，包含item2owners,peers2nonces,nonces2peers，acceptingDigests，acceptingResponses等属性

```golang
type PullEngine struct {
	PullAdapter
	stopFlag 			int32
	state 				*util.Set
	item2owners 		map[string][]string
	peers2nonces 		map[string]uint64
	nonces2peers 		map[uint64]string
	acceptingDigests 	int32
	acceptingResponses 	int32
	lock 				sync.Mutex
	outgoingNONCES 		*util.Set
	incomingNONCES 		*util.Set
	digFilter 			DigestFilter
}
```

## 构造方法

### NewPullEngineWithFilter方法

创建一个PullEngine实例，传入一个再pull初始化的休眠时间，和发送digests和response消息时用到的filter

参数：

- participant PullAdapter：PullAdapter实例
- sleepTime time.Duration：在pull初始化之间的休眠时间
- df DigestFilter：过滤器

返回值：

- *PullEngine

首先，创建一个PullEngine，然后新启一个go routine, 执行engine.initiatePull()初始化pull协议

### NewPullEngine方法

新建一个PullEngine实例，不传入filter

参数：

- participant PullAdapter：PullAdapter实例
- sleepTime time.Duration：在初始化之前休眠时间

返回值：

- *PullEngine

调用NewPullEngineWithFilter创建实例，其中filter为全返回true的方法

## 普通方法

### initiatePull方法

初始化pull协议

调用engine.acceptDigests()方法接受digests。对于engine.SelectPeers中的每个peer，调用engine.newNonce创建nonce，将nonce添加到engine.outgoingNONCES，在engine.nonces2peers和engine.peers2nonces中建立映射，调用engine.Hello(peer, nonce)发送hello消息

调用等待digest回应，如果收到回应调用engine.processIncomingDigests方法

### acceptDigests方法

将engine.acceptingDigests设置为1

### newNonce方法

调用util.RandomUInt64()方法产生一个以前没有产生过的随机数

### processIncomingDigests方法

处理收到的Digests

首先，调用engine.ignoreDigests()方法将engine.acceptingDigests设置为0.

创建一个string到[]string的map,存储每个节点的item，将item2owner转换到requestMapping中。

调用engine.acceptReponses()接受回应.

对于requestMapping中的每个条目，发送engine.SendReq(dest, seqsToReq, engine.peers2nonces[dest])

等待responseWaitTime等待回应，如果收到消息调用engine.endPull结束Pull协议

### ignoreDigests方法

将engine.acceptingDigests设置为0

### acceptResponse方法

将engine.acceptingResponses设置为1

### endPull方法

首先将engine.acceptingResponses设置为0，清空engine.outgoingNONCES，engine.item2owners, engine.peers2nonces和engine.nonces2peers

### toDie方法

将engine.stopFlag设置为1

### isAcceptingResponses方法

返回engine.accepingResponse是否为1

### isAcceptingDigests方法

返回engine.acceptingDigests是否为1

### Stop方法

将engine.stopFlag设置为1

### OnDigest方法

通知engine实例一个digest到达

参数：

- digest []string
- nonce uint64
- context interface{}

首先，如果!engine.isAcceptingDigests()或者!engine.outgoingNONCES.Exists(nonce)，直接返回

对于digests中的条目，将其和peer添加到item2owner。

### Add方法

将请求中的item添加到engine.state中

参数：

- seqs ...string：请求

### Remove方法

将请求中的item从engine.state中移除

### OnHello方法

当hello消息到达时通知engine

参数：

- nonce uint64
- context interface{}

首先将nonce添加到engine.incomingNONCES中

等待reqest，当request到达时调用engine.incomingNONCES.Remove(nonce)将nonce移除

对于engine中的state，用engine.digFilter(context)进行过滤。

最后调用engine.SendDigest(digest, nonce, context)发送digest

### OnReq方法

通知engine有request到达

参数：

- items []string
- nonce uint64
- context interface{}

如果!engine.incomingNONCES.Exists(nonce)返回

通过engine.digFilter(context)过滤掉items的条目

新启go routine，调用engine.SendRes(items2Send, context, nonce)发送回应

### OnRes方法

OnRes通知engine收到了一个回应

```golang
if !engine.outgoingNONCES.Exists(nonce) || !engine.isAcceptingResponses() {
	return
}
engine.Add(items...)
```
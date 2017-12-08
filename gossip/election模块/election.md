election
===

Gossip/election模块实现leader选举，算法属性：

- 通过比较ID打破symmetry
- 节点时leader或者是follower。如果对于所有节点的成员视图相同，目标是仅选择一个leader
- 如果网络分成2个或者多个分区，那么leader的数量就是分区的数量，当分区消失时，那么久只剩一个分区
- Peers通过gossip leadership proposal或declaration消息进行通信

算法的伪代码：

```
变量：
	leaderKnown=false
不变式：
	peer监听远端节点的消息，无论何时接收到leadership declaration消息，leaderKnown则被设置为true
Startup():
	等待membership稳定，或者收到leadership declaration消息，或者startup超时。跳转到SteadyState()
SteadyState():
	while true:
		如果leaderKnown是false：
			LeaderElection()
		如果节点是leader：
			广播leadership declaration消息
			如果收到拥有更低的ID的peer的leadership declaration消息，成为follower
		如果节点是false：
			在一定时间内没有收到leadership declaration消息：
				将leaderKnow设置为false
LeaderElection()：
	Gossip传输leadership proposal消息
	收集一段时间内其他节点发送的消息
	如果收到一个leadership declaration消息：
		返回
	遍历所有收集的proposal消息，如果有消息来自更低ID的节点，返回。否则declare自己为leader
```	

# 全局方法及类型定义

## leadershipCallback类型

leadership的回调函数

```golang
type leadershipCallback func(isLeader bool)
```

## peerID类型

字节数组

```golang
type peerID []byte
```

## 全局方法

- SetStartupGracePeriod: 设置startup阶段的等待时间
- SetMembershipSampleInterval：设置检查membership视图的频率
- SetLeaderAliveThreshold：设置leader election alive时间
- SetLeaderElectionDuration:设置等待leader选举完成的时间
- getStartupGracePeriod
- getMembershipSampleInterval
- getLeaderAliveThreshold
- getLeaderElectionDuration
- GetMsgExpirationTimeout

# LeaderElectionAdapter接口

LeaderElectionAdapter被leader选举模块用来发送和接受消息，以获取成员信息

```golang
type LeaderElectionAdapter interface {
	Gossip(Msg)
	Accept() <-chan Msg
	CreateMessage(isDeclaration bool) Msg
	Peers() []Peer
}
```

# Peer接口

Peer接口描述一个远端peer

```golang
type Peer interface {
	ID() peerID
}
```

# Msg接口

Msg接口描述了远端节点发送的消息

```golang
type Msg interface {
	SenderID() peerID
	IsProposal() bool
	IsDeclaration() bool
}
```

# LeaderElectionService接口

LeaderElectionService是运行leader选举算法的接口

```golang
type LeaderElectionService interface {
	IsLeader() bool
	Stop()
	Yield()
}
```

# leaderElectionSvcImpl类

实现LeaderElectionService接口

## 属性

leaderElectionSvcImpl类包含id, proposals, isLeader, leaderExists, adapter, callback等属性

```golang
type leaderElectionSvcImpl struct {
	id peerID
	proposals *util.Set
	sync.Mutex
	stopChan chan struct{}
	interruptChan chan struct{}
	stopWG sync.WaitGroup
	isLeader int32
	toDie int32
	leaderExists int32
	yield int32
	sleeping bool
	adapter LeaderElectionAdapter
	logger *logging.Logger
	callback leadershipCallback
	yieldTimer *time.Timer
}
```

## 构造函数

### NewLeaderElectionService方法

返回一个新的LeaderElectionService

- adapter LeaderElectionAdapter: 
- id string
- callback leadershipCallback

返回值：

- LeaderElectionService

创建一个leaderElectionSvcImpl。新启线程，调用le.start()。返回le

## 普通方法

### start方法

开始leader选举算法

首先，将stopWG加2。

新启线程，调用le.handleMessages()处理消息。

调用le.waitFromMembershipStabilization()等待membership视图稳定

新启线程，调用le.run运行算法

### handleMessage方法

首先调用le.adapter.Accept()方法获取消息通道msgChan。

处理通道信号。如果是le.stopChan的信号，将空对象发送到le.stopChan中，并返回。如果是msgChan中的信号，如果!le.isAlive(msg.SenderID())，跳转到下一次循环，否则调用le.handleMessage(msg)处理消息。

### isAlive方法

返回给定的peer是否被认为是alive。

遍历le.adapter.Peers()获得的Peer，如果有peer的id与给定的peerID相同，返回真；否则返回假。

### handleMessage方法

首先判断msgType是proposal还是declaration。

如果是proposal消息，将ID添加到le.proposals中；

如果是declaration消息，将le.leaderExists设置为1。如果le.sleeping&&len(le.interruptChan)==0，将空对象发送到le.interruptChan中。如果消息发送者的id小于节点自身的ID，且节点是leader，调用le.stopBeingLeader()不再成为leader。

### IsLeader方法

返回节点是否为leader

### stopBeingLeader方法

将le.isLeader设置为0。调用回调函数le.callback(true)通知节点不再是leader

### waitForMembershipStabilization方法

等待membership视图稳定，或者超过一定时限，或者一个节点宣称自己是leader

视图稳定是指在一定时间(getMembershipSampleInterval())内viewSize=len(le.adapter.Peers())不再改变。

```golang
endTime := time.Now().Add(timeLimit)
viewSize := len(le.adapter.Peers())
for !le.shouldStop() {
	time.Sleep(getMembershipSampleInterval())
	newSize := len(le.adapter.Peers())
	if newSize == viewSize || time.Now().After(endTime) || le.isLeaderExists() {
		return
	}
	viewSize = newSize
}
```

### isLeaderExists方法

返回le.leaderExists是否为1

### run方法

运行选举算法

如果没有退出选举，调用le.leaderElection方法进行选举;

```golang
if !le.isLeaderExists() {
	le.leaderElection()
}
```

如果其他节点被选举且节点自己在yielding，调用le.stopYielding()停止yielding

```golang
if le.isLeaderExists() && le.isYielding() {
	le.stopYielding()
}
```

如果选举算法停止，直接返回；

```golang
if le.shouldStop() {
	return
}
```

如果自己是leader，调用le.leader();否则调用le.follower

```golang
if le.IsLeader() {
	le.leader()
} else {
	le.follower()
}
```

### shouldStop方法

返回le.toDie是否为1

### isYielding方法

返回le.yield是否为1

### stopYielding方法

将le.yielding设置为0，停止le.yieldTimer

### leaderElection方法

进行leader选举

如果le.isYielding()，说明节点正在放弃成为leader，直接返回

调用le.propose()方法，将自己提议为leader，调用le.waitForInterrupt收集其他的proposal

如果le.isLeaderExists()，说明其他节点宣称自己是leader，返回

如果le.isYielding()，返回

遍历收集到的proposal，如果有节点的id小于自己的id，返回。否则，调用le.beLeader()方法宣称自己是leader，并将le.leaderExists设置为1

### propose方法

发送一个leadership提议到远端节点

首先，调用le.adapter.CreateMessage(false)方法创建一个proposal消息，调用le.adapter.Gossip()方法将消息gossip到远端节点

### waitForInterrupt方法

waitForInterrupt方法一直休眠到interrupt通道被触发，或者给定的时间超时

首先将le.sleepingg设置为true。对于不同通道的信号进行处理

```golang
select {
	case <-le.interruptChan:
	case <-le.stopChan:
		le.stopChan <-struct{}{}
	case <-time.After(timeout):
}
```

将le.sleeping设置为false。调用drainInterruptChannel()清理interrupt通道，以防在休眠时收到2个leadership declarations消息

### drainInterruptChannel方法

如果len(le.interruptChan)==1，将消息排出去

```golang
if len(le.interruptChan) == 1 {
	<-le.interruptChan
}
```

### beLeader方法

将le.isLeader设置为1，调用le.callback(true)方法处理回调函数

### leader方法

创建declaration消息，调用le.adapter.Gossip发送到远端节点

然后调用le.waitForInterrupt休眠

### follower消息

清空le.proposals，将le.leaderExists设置为0

处理信号

```golang
select {
	case <-time.After(getLeaderAliveThreshold()):
	case <-le.stopChan:
		le.stopChan <- struct{}{}
}
``` 

### Yield方法

放弃成为leader

如果!le.IsLeader() || le.isYielding()，直接返回

调用stopBeingLeader停止成为leader，将le.leaderExists这是为0.

### Stop方法

将le.toDie设置为1，向le.stopChan中发送空对象，调用le.stopWG等待其他线程结束。
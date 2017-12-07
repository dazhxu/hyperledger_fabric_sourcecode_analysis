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
	等待membership稳定，或者
```	
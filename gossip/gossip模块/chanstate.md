chanstate
===

chanstate定义了channelState结构和gossipAdapterImpl结构

# channelState结构

存储通道状态

## 属性

包括有string到channel.GossipChannel的映射和一个gossipServiceImpl

```golang
type channelStae struct {
	stopping int32
	sync.RWMutex
	channels map[string]channel.GossipChannel
	g *gossipServiceImpl
}
```

## 方法

### stop方法

停止channelState。首先将cs.stopping设置为1，针对cs.channels中的每个条目gc，调用gc.Stop停止通道

### isStopping方法

返回cs.stopping是否为1

### lookupChannelForMsg方法

为消息寻找发送的channel

参数：

- msg proto.ReceivedMessage

返回值：

- channel.GossipChannel

如果消息是StateInfoPullRequestMsg，调用cs.getGossipChannelByMAC获取通道。

```golang
if msg.GetGossipMessage().IsStateInfoPullRequestMsg() {
	sipr := msg.GetGossipMessage.GetStateInfoPullReq()
	mac := sipr.Channel_MAC
	pkiID := msg.GetConnectionInfo().ID
	return cs.getGossipChannelByMAC(mac, pkiID)
}
```

否则，调用cs.lookupChannelForGossipMsg为gossip消息寻找channel

### getGossipChannelByMAC方法

遍历cs.channels，针对每个channel生成mac，如果mac与收到的mac相同，则返回此channel；如果找不到，则返回nil

### lookupChannelForGossipMsg方法

如果!msg.IsStateInfoMsg，说明消息不是StateInfoPullRequest，也不是StateInfo消息。因为，过去已经发给此节点StateInfo消息，证明它已经知道channel名字，因此调用cs.getGossipChannelByChainID(msg.Channel)直接通过消息自身获取channel名字。

否则，说明消息是StateInfo消息，从消息中获取stateInfo，调用cs.getGossipChannelByMAC获取通道名

```golang
stateInfMsg := msg.GetStateInfo()
return cs.getGossipChannelByMAC(stateInfMsg.Channel_MAC, stateInfMsg.PkiId)
```

### getGossipChannelByID方法

从cs.channels中直接获取channel名字

### joinChannel方法

将此节点加入channel

参数：

- joinMsg api.JoinChannelMessage
- chainID common.ChainID

如果cs.channels中存在ChainID，调用gc.ConfigChannel配置channel；如果cs.channels不存在此chainID, 创建一个GossipChannel，并加入cs.channels

```golang
if gc, exists := cs.channels[string(chainID)]; !exists {
	pkiID := cs.g.comm.GetPKIid()
	ga := *gossipAdapterImpl(gossipSeriveImpl: cs.g, Doscovery: cs.g.disc)
	gc := channel.NewGossipChannel(pkiID, cs.g.selfOrg, cs.g.mcs, chainID, ga, joinMsg)
	cs.channels[string(chainID)] = gc
} else {
	gc.ConfigureChannel(joinMsg)
}
```

# gossipAdapterImpl结构

## 属性

gossipAdapterImpl继承gossipServiceImpl和discovery.Discovery属性

```golang
type gossipAdapterImpl struct {
	*gossipServiceImpl
	discovery.Discovery
}
```

## 方法

### GetConf方法

调用ga.conf的属性，创建channel.Conf

```golang
return channel.Config {
	ID:                          ga.conf.ID,
	MaxBlockCountToStore:        ga.conf.MaxBlockCountToStore,
	PublishStateInfoInterval:    ga.conf.PublishStateInfoInterval,
	PullInterval:                ga.conf.PullInterval,
	PullPeerNum:                 ga.conf.PullPeerNum,
	RequestStateInfoInterval:    ga.conf.RequestStateInfoInterval,
	BlockExpirationInterval:     ga.conf.PullInterval * 100,
	StateInfoCacheSweepInterval: ga.conf.PullInterval * 5,
}
```

### Gossip方法

调用ga.gossipServiceImpl.emmitter.Add(msg)将消息添加到发送器

### Send方法

调用ga.gossipServiceImpl.comm.Send(msg, peers...)发送消息

### ValidateaStateInfoMessage方法

调用ga.gossipServiceImpl.validateStateInfoMsg(msg)验证消息

### GetOrgOfPeer方法

调用ga.gossipServiceImpl.getOrgOfPeer(PKIID)返回特定PKIID的组织

### GetIdentityByPKIID方法

返回特定PKIID节点的identity
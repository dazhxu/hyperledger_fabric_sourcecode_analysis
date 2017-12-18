gossip_impl
===

gossip_impl定义了gossipServiceImpl结构及相关的结构，是gossip层的实现

# 常量、全局变量与类型定义

## 常量

```golang
const (
	presumedDeadChanSize = 100 //被认为是断开的chan的大小
	accpectChanSize      = 100 //可接收的chan的大小
)
```

## 变量

```golang
identityExpirationCheckInterval = time.Hour * 24 //每24小时检查失效的节点
identityInactivityCheckInterval = time.Minute * 10 //每10分钟检查不活跃的身份
```

## channelRoutingFilterFactory函数类型

用来创建filter.RoutingFilter的工厂方法

```golang
type channelRoutingFilterFactory func(channel.GossipChannel) filter.RoutingFilter
```

# discoveryAdapter结构

dicoveryAdapter为discovery模块提供其需要的通信接口

## 属性


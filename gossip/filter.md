filter
===

filter模块定义了与过滤器相关的类型定义和方法

# 变量与函数类型

## RoutingFilter函数类型

RoutingFilter定义了NetworkMember上的判定。来判断给定的NetworkMember是不是应该被选择来被发送一个消息

```golang
type RoutingFilter func(discovery.NetworkMember) bool
```

## SelectNonPolicy函数变量

选择空的成员

```golang
var SelectNonePolicy = func(discovery.NetworkMember) bool {
	return true
}
```

## SelectAllPolicy函数变量

选择所有的成员

```golang
var SelectAllPolicy = func(discovery.NetworkMember) bool {
	return true
}
```

# 全局方法

## CombineRoutingFilters方法

用逻辑与方法将多个给定的routing filter结合起来

```golang
return func(member discovery.NetworkMember) bool {
	for _, filter := range filters {
		if !filter(member) {
			return false
		}
	}
	return true
}
```

## SelectPeers方法

选择满足routing filter的一组peer

参数：

- k int: 如果满足条件的节点个数大于k，则选择k个；否则返回所有满足添加的节点
- peerPool []discovery.NetworkMember
- filter RoutingFilter

返回值：

- []*comm.RemotePeer
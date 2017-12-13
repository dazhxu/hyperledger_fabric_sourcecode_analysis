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

PullEngine需要PullAdapter
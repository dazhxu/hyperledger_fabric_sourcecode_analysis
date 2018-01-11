server
===

server文件定一个orderer的服务功能

# configUpdateSupport结构

支持configUpdate

## 属性

```golang
type configUpdateSupport struct {
	multichain.Manager
}
```

## 方法

### GetChain方法

调用cus.Manager.GetChain(chainID)获取一个configupdate.Support对象

# broadcastSupport结构

支持broadcast

## 属性

包含multichain.Manager和broadcast.ConfigUpdaeProcessor

```golang
multichain.Manager
broadcast.ConfigUpdateProcessor
```

## 方法

### GetChain方法

调用bs.Manager.GetChain方法，返回一个broadcast.Support对象

# deliverSupport对象

## 属性

包含multichain.Manager属性

```golang
type deliverSupport struct {
	multichain.Manager
}
```

## 方法

### GetChain方法

调用Manager.GetChain方法返回一个deliver.Support对象

# server类

## 属性

包含一个broadcast.Handler和一个deliver.Handler处理broadcast和deliver

```golang
type server struct {
	bh bradcast.Handler
	dh deliver.Handler
}
```

## 构造方法

### NewServer方法

创建一个server对象

```golang
func NewServer(ml multichain.Manager, signer crypto.LocalSigner) ab.AtomicBroadcastServer {
	s := &server{
		dh: deliver.NewHandlerImpl(deliverSupport{Manager: ml}),
		bh: broadcast.NewHandlerImpl(broadcastSupport{
			Manager: ml,
			ConfigUpdateProcessor: configupdate.New(ml.SystemChannelID(), configUpdateSupport{Manager: ml}, signer),
		}),
	}
	return s
}
```

## 普通方法

### Broadcast方法

Broadcast从client接受要排序的消息流

参数：

- srv ab.AtomicBroadcast_BroadcastServer：广播消息流

返回值：

- error

调用s.bh.Handle(srv)方法

### Deliver方法

排序后发送一个block的流给peer

参数：

- srv ab.AtomicBraodcast_DeliverServer

返回值：

- error

调用s.dh.Handle(srv)处理deliver功能
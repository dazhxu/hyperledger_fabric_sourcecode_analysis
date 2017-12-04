conn
===

conn子模块主要定义了connection对象和connectionStore对象，connection指与远端节点的连接，connectionStore存储了到远端节点的所有连接。

# msgSending对象

定义了发送的数据。

```golang
type msgSending struct {
	envelope *proto.Envelope
	onErr func(error)
}
```

# connection对象

## 属性

connection对象包含了缓存通道、节点身份、连接、客户端stream、服务器stream等属性

```golang
type connection struct {
	cancel			context.CancelFunc
	info			*proto.ConnectionInfo
	outBuff			chan *msgSending
	logger			*logging.Logger
	pkiID 			common.PKIidType
	handler			handler
	conn 			*grpc.ClientConn
	cl 				proto.GossipClient
	clientStream	proto.Gossip_GossipStreamClient
	serverStream 	proto.Gossip_GossipStreamServer
	stopFlag 		int32
	stopChan 		chan struct{}
	sync.RWMutex
}
```

其中：

- outBuff：peer的发送缓存
- pkiID：common.PKIidType
- handler: 接受到消息调用的方法
- conn：到远端节点的gRPC连接
- cl: 远端节点的grpc stub
- clientStream：到远端节点的客户端侧的stream
- serverStream：到远端节点的服务端侧的stream

## 构造函数

### newConnection方法

新建一个connection

参数：

- cl proto.GossipClient: 远端节点的grpc stub
- c *grpc.ClientConn: 到远端节点的grpc连接
- cs proto.Gossip_GossipStreamClient: 到远端节点的客户端侧的stream
- ss proto.Gossip_GossipStreamServer: 到远端节点的服务器测的stream

返回值：

- *connection

## 普通方法

### close方法

关闭此connection。首先，向stopChan通道发送一个空对象；然后上锁，调用conn.clientStream.CloseSend()关闭发送；调用conn.conn.Close()关闭连接；调用conn.cancel()关闭；解锁

### toDie方法

返回stopFlag是否为1

### send方法

发送签名消息。

参数：

- msg *proto.SignedGossipMessage: 发送的消息
- onErr func(error): 错误处理函数

首先，加锁，保证操作的互斥性；判断发送缓存是否满，若满，直接返回；新建msgSending对象，并输出到outBuff通道。

```golang
m := &msgSending {
	envelop: msg.Envelope,
	OnErr: onErr,
}
conn.outBuff <- m
```

### serviceConnection方法

提供链接服务。

首先创建两个通道，errChan用于在进程间传递错误，msgChan用于在进程间传递消息。

新建线程，执行conn.readFromStream(errChan, msgChan)方法。在readFromStream方法中异步调用stream.Recv()方法，并等待Recv()方法介绍，或者关闭连接的信号。

新建线程，执行conn.writeToStream()方法。

在原线程中，处理通道中的信号。如果是stopChan中的消息，从新发送给stopChan；如果是errChan中的消息，返回err；如果是msgChan的消息，调用conn.handler(msg)处理消息。

```golang
for !conn.toDie() {
	select {
	case stop := <-conn.stopChan:
		conn.stopChan <- stop
		return nil
	case err := <-errChan:
		return err
	case msg := <-msgChan:
		conn.handler(msg)
	}
}
```

### writeToStream方法

将outBuff通道的消息进行写入stream，并调用stream.Send(m.envelope)方法发送。

首先调用getStream方法获取stream。然后针对不同的信息进行处理。如果是outBuff中的通道，调用stream.Send(m.envelope)发送消息，出现错误时，调用m.onErr(err)处理消息。

```golang
select {
case m := <-conn.outBuff:
	err := stream.Send(m.envelope)
	if err != nil {
		go m.onErr(err)
		return
	}
case stop := <-conn.stopChan:
	conn.stopChan <- stop
	return
}
```

### readFromStream方法

从stream中接受消息，并发送到msgChan通道中

参数：

- errChan chan error: 错误通道
- msgChan chan *proto.SignedGossipMessage: 消息通道

首先，定义recover()，在msgChan关闭是捕获panic

```golang
defer func() {
	recover()
}()
```

然后，获取stream，并调用stream.Recv()方法获取消息。

```golang
stream := conn.getStream()
if stream == nil {
	errChan <- errors.New("Stream is nil!")
	return
}
envelope, err := stream.Recv()
if conn.toDie() {
	return
}
if err != nil {
	errChan <- err
	return
}
```

最后，将envelop转换为GossipMessage并发送到msgChan中

```golang
msg, err := envelope.ToGossipMessage()
if err != nil {
	errChan <- err
}
msgChan <- msg
```

### getStream方法

返回stream。如果conn.clientStream和conn.serverStream都不为空，返回错误。返回conn.clientStream或conn.serverStream.

# connectionStore对象

connectionStore对象管理peer到远端节点的连接

## 属性

```golang
type connectionStore struct {
	logger 				*logging.Logger
	isClosing			bool
	connFactory 		connFactory
	sync.RWMutex 
	pki2Conn 			map[string]*connection
	destinationLocks 	map[string]*syncRWMutex
}
```

其中

- connFactory connFactory：创建到远端节点的连接的工厂类
- pki2Conn map[string]*connection：定义了从pkiid到连接的映射
- destinationLocks map[string]*syncRWMutex: 定义了从pkiid到互斥锁的映射


## 构造方法

### newConnStore方法

新建一个connectionStore。

参数：

- connFactory connFactory: 创建connection的工厂类
- logger *logging.Logger

## 普通方法

### getConnection方法

根据peer获取连接，如果pki2Conn中存在此连接，直接返回；否则，新建一个连接。

参数：

- peer *RemotePeer: 远端节点

返回值：

- *connection
- error

### connNum方法

返回连接的数量

### closeConn方法

关闭连接

### shutdown方法

关闭

```golang
wg := sync.WaitGroup{}
for _, conn := range connections2Close {
	wg.Add(1)
	go func(conn *connection) {
		cs.closeByPKIid(conn.pkiID)
		wg.Done()
	} (conn)
	wg.Wait()
}
```

### onConnected方法

在连接时，进行的动作

参数：

- serverStream proto.Gossip_GossipStreamServer
- connInfo *proto.ConnectionInfo

返回值：

- *connection

如果连接存在，则关闭连接

调用cs.registerConn方法注册连接

### registerConn方法

注册连接。即新建连接，初始化连接中的各个属性

### closeByPKIid方法

根据pkiID关闭连接
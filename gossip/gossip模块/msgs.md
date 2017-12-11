msgs
===

msgs用于存储消息

# 类型定义与全局函数

## invalidationTrigger类型

在每个消息因添加需要被验证时，invalidationTrigger被调用。例如，add(0)和add(1)依次被调用，在调用结束后store最终的状态只是{1}。

```golang
type invalidationTrigger func(message interface{})
```

## msg结构

msg定义了消息的格式

```golang
type msg struct {
	data interface{}
	created time.Time
	expired bool
}
```

# MessageStore接口

Message接口添加消息到内部的缓存。当收到一个消息时，它可能会：

- 添加到缓存
- 因为其他一些已经存在于缓存中，忽略此消息
- 使已经在缓存中的消息失效

当验证一个消息时，invalidationTrigger在这个消息上被调用

```golang
type MessageStore interface {
	Add(msg interface{}) bool
	CheckValid(msg interface{}) bool
	Size() int
	Get() []interface{}
	Stop()
	Purge(func(interface{}) bool)
}
```

- Add方法：添加一个消息到store中
- CheckValid方法：检查消息是否能有效地插入store
- Size方法：返回store中消息的数量
- Get方法：返回store中所有的消息
- Stop方法：停止所有的go routines
- Purge方法：根据预设条件清除所有消息

# messageStoreImpl对象

messageStoreImpl是对MessageStore接口的实现

## 属性

messageStoreImpl包含messages、msgTTL、invTrigger等属性

```golang
type messageStoreImpl struct {
	pol 				commom.MessageReplacingPolicy
	lock 				sync.RWMutex
	messages 			[]*msg
	invTrigger 			invalidationTrigger
	msgTTL 				time.Duration
	expiredCount 		int
	externalLock		func()
	externalUnlock 		func()
	expireMsgCallback 	func(msg interface{})
	doneCh 				chan struct{}
	stopOnce 			sync.Once
}
```

## 构造方法

### NewMessageStore方法

NewMessageStore方法传入消息替换策略和有效性检验触发器，返回一个新的Message Store

```golang
func NewMessageStore(pol common.MessageReplacingPolicy, trigger invalidationTrigger) {
	return newMessageStore(pol, trigger)
}
```

### NewMessageStoreExpirable方法

创建新的MessageStore，支持在msgTTL后旧消息失效。在失效时，首先externalLock加锁，expiraiton回调函数被调用，最后externalUnlock

参数：

- pol common.MessageReplacingPolicy
- trigger invalidationTrigger
- msgTTL time.Duration
- externalLock func()
- externalUnlock func()
- externalExpire func(interface{})

返回值：

- MessageStore

首先，调用newMsgStore(pol, trigger)创建一个新的msgstore；然后设置externalLock、externalUnlock、externalExpire。启动一个go routine调用store.expirationRoutine方法

### newMsgStore方法

创建一个MessageStore

## 其他方法

### expirationRoutine方法

失效routine，针对不同通道的信号进行处理。如果是s.doneCh的信号，返回；如果是time.After(s.expirationCheckInterval())的信号，创建hasMessageExpired判断器，如果s.isPurgeNeeded(hasMessageExpired)为真，调用s.expireMessages失效消息。

```golang
case <-time.After(s.expirationCheckInterval()):
	hasMessageExpired := func(m *msg) bool {
		if !m.expired && time.Since(m.created) > s.msgTTL {
			return true
		} else if time.Since(m.created) > (s.msgTTL * 2) {
			return true
		}
	}
	if s.isPurgeNeeded(hasMessageExpired) {
		s.expiredMessages()
	}
```

### isPurgeNeeded方法

是不是需要执行清理消息

参数：

- shouldBePurged func(*msg) bool

对于messages中的每条消息，如果有消息shouldBePurged(m)，返回true。否则返回false

### expireMessages方法

失效消息。

对于每个消息，如果!m.expired，如果time.Since(m.created) > s.msgTTL, 将m.expired设置为true，调用s.expireMsgCallback(m.data)执行回调函数，s.expiredCount++; 如果m.expired为真，且time.Since(m.created) > (s.msgTTL *2)，将s.messages[i]移除

```golang
if !m.expired {
	if time.Since(m.created) > s.msgTTL {
		m.expired = true
		s.expireMsgCallback(m.data)
		s.expiredCount++
	}
} else {
	if time.Since(m.created) > (s.msgTTL * 2) {
		s.messages = append(s.messages[:i], s.messages[i+1:]...)
		n--
		i--
		s.expiredCount--
	}

}
```

### Add方法

将一个消息添加到store

对于s.messages中的每个消息m，如果s.pol(message, m.data)为common.MessageInvalidated，返回false；如果为common.MessageInvalidates, 调用s.invTrigger(m.data)验证消息，并将其从message中删除

根据message创建msg对象，并将其添加到s.messages中

### Purge方法

清除消息.

对于每个需要清除的消息，先调用s.invTrigger(s.messages[i].data)，然后将其从s.messages中删除

### CheckValid方法

检查消息是否能有效插入store中

对于s.messages中的每个消息，如果有消息使s.pol(message, m.data) == common.MessageInvalidated，返回false；否则返回true

### Size方法

返回有效消息的大小

### Get方法

返回有效的消息

### Stop方法

创建stopFunc，关闭s.doneCh

调用s.stopOnce.Do(stopFunc)


### expirationCheckInterval方法

返回s.msgTTL/100
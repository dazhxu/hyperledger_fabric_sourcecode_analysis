chain
===

# chainImpl对象

chainImpl类实现了orderer的链功能。

## 属性

chainImpl类包含consenter、ConsenterSupport、channel等属性。

```golang
type chainImpl struct {
	consenter commonConsenter
	support multichain.ConsenterSupport
	channel channel
	lastOffsetPersisted int64
	lastCutBlockNumber uint64
	producer sarama.SyncProducer
	parentConsumer sarama.Consumer
	channelConsumer sarama.PartitionConsumer

	errorChan chan struct{}
	haltChan chan struct{}
	startChan chan struct{}
}
```

## 构造方法

### newChain方法

创建并返回chainImpl对象

首先调用getLastCutBlockNumber(support.Height())方法获取上个区块高度；创建errorChan，并关闭。

返回chainImpl对象

参数：

- consenter commonConsenter
- support multichain.ConsenterSupport
- lastOffsetPersisted int64

返回值：

- *chainImpl
- error

## 普通方法

### Errord方法

返回一个通道，当分区消费者发送错误时，关闭通道。

### Start方法

Start方法为保持最新的链而分配必要的资源。实现multichain.Chain接口。被multichain.NewManagerImpl()方法调用，后者在orderer进程启动是被调用。

新启go routine，执行startThread方法，启动chain。

### startThread方法

首先会调用setupProducerForChannel，设置生产者；然后调用sendConnectMessage发送连接消息；然后调用setupParentConsumerForChannel设置parent消费者；调用setupChannelConsumerForChannel设置通道消费者；创建errorChan，调用chain.processMessagesToBlocks，处理消息，创建区块。

### setupProducerForChannel方法

利用给定的retry选项，为通道设置生产者。

参数：

- retryOptions localconfig.Retry
- haltChan chan struct{}
- brokerConfig *sarama.Config
- channel channel

返回值：

- sarama.SyncProducer
- error

调用sarama.NewSyncProducer()方法创建生产者。

```golang
setupProducer := newRetryProcess(retryOptions, haltChan, channel, retryMsg, func() error {
	producer, err = sarama.NewSyncProducer(brokers, brokerConfig)
	return err
})
```

### sendConnectMessage方法

利用给定的retry选项，向通道发送CONNECT消息。

参数：

- retryOptions localconfig.Retry
- exitChan chan struct{}
- producer sarama.SyncProducer
- channel channel

返回值：

- error

调用newConnectMessage方法创建连接消息，然后创建Producer消息。

```golang
payload := utils.MarshalOrPanic(newConnectMessage())
message := newProducerMessage(channel, payload)
```

调用producer.SendMessage发送消息

```golang
postConnect := newRetryProcess(retryOptions, exitChan, channel, retryMsg, func() error {
	_, _, err := producer.SendMessage(message)
	return err
})
```

### newConnectMessage方法

创建内容为空的连接消息，返回*ab.KafkaMessage

```golang
return &ab.KafkaMessage_Connect{
	Connect: &ab.KafkaMessageConnect{
		Payload: nil,
	},
}
```

### newProducerMessage方法

创建一个sarama.ProducerMessage消息

### setupParantConsumerForChannel

利用给定的retry选项设置parent consumer

```golang
setupParentConsumer := newRetryProcess(retryOption, haltChan, channel, retryMsg, func() error {
	parentConsumer, err = sarama.NewConsumer(brokers,brokerConfig)
	return err
})
```

### setupChannelConsumerForChannel方法

利用给定的retry选项设置通道consumer

```golang
setupChannelConsumer := newRetryProcess(retryOptions, haltChan, channel, retryMsg, func() error {
	channelConsumer, err = parentConsumer.ConsumerPartition(channel.topic(), channel.partition(), startFrom)
})
```

### processMessagesToBlocks方法

利用给定通道的Kafka consumer，负责将ordered消息流转换为通道账本的区块。

如果是chain.haltChan，将counts[indexExitChanPass]++并返回；

如果是chain.channelConsumer.Errors()信号，说明在消费消息是发生错误，关闭通道。新建go routine,调用sendConnetionMessage方法，进行重试。

如果是chain.channelConsumer.Messages()信号，如果发生错误，返回。如果是chain.errorChan信号，即channel被关闭，创建一个新的channel。然后，将数据Unmarshal成消息。然后对消息类型进行判断。如果ab.KafkaMessage_Connect消息，调用processConnection(chain.support.ChainID())进行处理；如果是ab.KafkaMessage_TimeToCut消息，调用processTimeToCut(msg.GetTimeToCut(), chain.support, &chain.lastCutBlockNumber, &timer, in.Offset)处理；如果是ab.KafkaMessage_Regular，调用processRegular处理。

如果是timer信号，调用sentTimeToCut消息。

### sendConnectMessage方法

使用给定的retry选项发送一个CONNECT消息到channel

首先创建Connect消息

```golang
payload := utils.MarshalOrPanic(newConnectMessage())
message := newProducerMessage(channel, payload)
```

创建重试函数，并发送消息到channel

```golang
postConnect := newRetryProcess(retryOptions, exitChan, channel, retryMsg, func() error {
	_, _, err := producer.SendMessage(message)
	return err
})
return postConnect.retry()
```

### processConnect方法

返回nil

### processTimeToCut方法

创建区块

参数：

- ttcMessage *ab.KafkaMessageTimeToCut: 
- support multichain.ConsenterSupport
- lastCutBlockNumber *uint64
- timer *<-chan time.Timer
- receivedOffset int64

返回值：

- error

首先，调用ttcMessage.GetBlockNumber获取区块高度。校验高度与lastCutBlockNumber+1是否一致。

调用support.BlockCutter().Cut()创建一个butch，调用support.CreateNextBlock方法创建区块，并将区块写入committer

```golang
batch, commiters := support.BlockCutter().Cut()
...
block := support.CreatNextBlock(batch)
encodedLastOffsetPersisted := utils.MarshalOrPanic(&ab.KafkaMetadata{LastOffsetPersisted: receivedOffset})
support.WriteBlock(block, committers, encodedLastOffsetPersisted)
```

### processRegular方法

处理regular消息

参数：

- regularMessage *ab.KafkaMessageRegular
- support multichain.ConsenterSupport
- timer *<-chan time.Time
- receivedOffset int64
- lastCutBlockNumber *uint64

首先将regularMessage.Payload数据转换为cb.Envelope。调用support.BlockCutter().Ordered(env)创建batch。如果batch长度为0且timer为空，创建一个timer并返回。

```golang
latches, committers, ok, pending := support.BlockCutter().Ordered(env)
if ok && len(batches) == 0 && *timer == nil {
	*timer = time.After(support.SharedConfig().BatchTimeout())
	return nil
}
```

如果最新的envelope不被包含在第一个batch里，第一个block的LastOffsetPersisted应该是receivedOffset-1

```golang
offset := receivedOffset
if pending || len(batches) == 2 {
	offset--
}
```

对于batches的每一个batch，创建区块并写入committers

```golang
for i, batch := range batches {
	block := support.CreateNextBlock(batch)
	encodedLastOffsetPersisted := utils.MarshalOrPanic(&ab.KafkaMetadata{LastOffsetPersisted: offset})
	support.WriteBlock(block, committers[i], encodedLastOffsetPersisted)
	*lastCutBlockNumber++
	offset++
}
```

如果batches不为空，将timer设置为nil

### sendTimeToCut方法

发送TimeToCut消息。

参数：

- producer sarama.SyncProducer
- channel channel
- timeToCutBlockNumber uint64
- timer *<-chan time.Time

返回值：

- error

创建payload，并发送

```golang
payload := utils.MarshalOrPanic(newTimeToCutMessage(timeToCutBlockNumber))
message := newProducerMessage(channel, payload)
_, _, err := producer.SendMessage(message)
return err
```

### Halt方法

释放分配给此链的资源。

```golang
close(chain.haltChan)
chain.closeKafkaObjects()
```

### Enqueue方法

接收一个消息，如果消息是可接受的，返回true，否则返回false

如果不是chain.haltChan的消息，说明通道没准备好，返回false

如果chain.haltChan的消息，说明链已经被停掉，返回false；

创建生产者消息并发送

```golang
marshaledEnv, err := utils.Marshal(env)
...
payload := utils.MarshalOrPanic(newRegularMessage(marshaledEnv))
message := newProducerMessage(chain.channel, payload)
_, _, err := chain.producer.SendMessage(message)
...
```

### closeKafkaObjects方法

关闭channel消费者、parent消费者和生产者

### getLastCutBlockNumber方法

返回blockchainHeight-1

### getLastOffsetPersisted方法

返回kafka的offset

参数：

- metadataValue []byte
- chainID string 

返回值：

- int64

如果metadataValue不为空，从metadatavalue中获取lastOffsetPersisted

```golang
kafkaMetadata := &ab.KafkaMetadata{}
err := proto.Unmarshal(metadataValue, kafkaMetadata)
...
return kafkaMetadata.LastOffsetPersisted
```

否则返回sarama.OffsetOldest - 1

### newConnectMessage方法

创建ab.KafkaMessage


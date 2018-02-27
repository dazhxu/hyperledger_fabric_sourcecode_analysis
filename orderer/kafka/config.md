config
===

# newBrokerConfig方法

创建broker的配置

- brokerConfig.Consumer.Retry.Backoff: 当读取kafka集群失败，重试前等待的时间
- brokerConfig.Consumer.Return.Errors: 允许读取在消费通道分区发生的错误
- brokerConfig.Metadata.Retry.Backoff: 读取metadata失败，重试前等待的时间
- brokerConfig.Metadata.Retry.Max: 重试最大次数
- brokerConfig.Net.DialTimeout: 连接超时时间
- brokerConfig.Net.ReadTimeout: 读操作的超时时间
- brokerConfig.Net.WriteTimeout: 写操作的超时时间
- brokerConfig.Net.TLS.Enable: tls选项
- brokerConfig.Net.TLS.Config: TLS配置
- brokerConfig.Producer.MaxMessageBytes: 生产者最大消息大小
- brokerConfig.Producer.Retry.Backoff: 生产者写数据失败后，重试前等待的时间
- brokerConfig.Producer.Retry.Max: 最大的重试次数
- brokerConfig.Producer.Partitioner: 生产者分区
- brokerConfig.producer.RequireAcks: 
- brokerConfig.Producer.Return.Success：
- brokerConfig.Version: kafka的版本
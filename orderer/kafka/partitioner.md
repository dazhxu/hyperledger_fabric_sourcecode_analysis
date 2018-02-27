partitioner
===

partitioner模块封装了sarama.Partitioner的部分功能

# staticPartitioner结构体

## 属性

包含partitionID属性

```golang
type staticPartitioner struct {
	partitionID int32
}
```

## 构造方法

### newStaticPartitioner方法

返回一个PartitionerConstructor

参数：

- partition int32

返回值：

- sarama.PartitionerConstructor

```golang
return func(topic string) sarama.Partitioner {
	return &staticPartitioner{partition}
}
```

## 普通方法

### Partition方法

接受一条消息和partition计数，返回partitionID

### RequiresConsistency方法

向partition的用户指明key->partition的映射是不是持久化的，返回true
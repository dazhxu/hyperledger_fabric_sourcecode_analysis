channel
===

# channel接口

channel标示基于kafka的orderer与之交互的kafka分区

```golang
type channel interface {
	topic() string
	partition() int32
	fmt.Stringer
}
```

# channelImpl结构

channelImpl是对channel的实现

## 属性

channelImpl的属性包括topic和partition

```golang
type channelImpl struct {
	tpc string
	prt int32
}
```

## 构造方法

### newChannel方法

返回一个channelImpl对象

## 普通方法

### topic方法

返回tpc属性

### partition方法

返回prt属性
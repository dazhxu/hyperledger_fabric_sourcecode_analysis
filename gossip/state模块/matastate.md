matastate
===

metastate模块定义了NodeMetastate类，存储账本当前高度信息

# NodeMetastate类

## 属性

```golang
type NodeMetastate struct {
	LedgerHeight uint64
}
```

## 构造函数

### NewNodeMetastate方法

参数为一个uint64，返回&NodeMetastate{height}

## 普通方法

### Bytes方法

将metadata序列化为bytes。首先显示声明数据格式为大端格式，将NodeMetadata写入缓存，调用buffer.Bytes()转换为[]byte

### Height方法

返回n.LedgerHeight

### Updata方法

更新n.LedgerHeight

# 其他方法

## FromBytes方法

将[]byte反序列化为NodeMetastate
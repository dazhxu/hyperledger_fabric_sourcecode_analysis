retry
===

定义retryProcess结构

# retryProcess结构

## 属性

包含轮询间隔、超时时间、通道、消息、处理函数等属性

```golang
type retryProcess struct {
	shortPollingInterval, shortTimeout time.Duration
	longPollingInterval, longTimeout time.Duration
	exit chan struct{}
	channel channel
	msg string
	fn func() error
}
```

## 构造方法

### newRetryProcess方法

接受localconfig.Retry参数和其他参数，返回一个retryProcess实例

## 普通方法

### retry方法

调用rp.try方法

```golang
if err := rp.try(rp.shortPollingInterval, rp.shortTimeout); err != nil {
	return rp.try(rp.longPollngInterval, rp.longTimeout)
}
```

### try方法

如果err:=rp.fn; err == nil，说明函数正常处理，直接返回nil。

否则，循环处理信号。如果是rp.exit信号，返回exitErr错误；如果是tickTotal.C返回错误；如果是tickInterval.C，重试，执行fn函数。
localconfig
===

localconfig定义了orderer用到的配置结构，具体的配置信息从orderer.yaml文件中读出

# init方法

初始化config模块

```golang
func init() {
	logger = flogging.MustGetLogger(pkgLogID)
	flogging.SetModuleLevel(pkgLogID, "error")

	configName = strings.ToLower(Prefix)
}
```

# 配置结构

## TopLevel结构

TopLevel对应orderer.yaml配置

```golang
type TopLevel struct {
	General 	General
	FileLedger 	FileLedger
	RAMLedger 	RAMLedger
	Kafka 		Kafka
}
```

## General结构

General中包含所有orderer类型共用的配置

```golang
type General struct {
	LedgerType     string
	ListenAddress  string
	ListenPort     uint16
	TLS            TLS
	GenesisMethod  string
	GenesisProfile string
	GenesisFile    string
	Profile        Profile
	LogLevel       string
	LogFormat      string
	LocalMSPDir    string
	LocalMSPID     string
	BCCSP          *bccsp.FactoryOpts
}
```

## TLS结构

TLS结构包含TLS连接的配置

## Profile结构

Profile结构包含go pprof profiling的配置

## FileLedger结构

FileLedger结构包含基于文件的账本的配置

## RAMLedger账本

RAMLedger包括基于RAM账本的配置

## Kafka结构

Kafka结构包括基于Kafka的orderer的配置

## Retry结构

Retry包括与重试和超时相关的配置，重试和超时因为不能和Kafka集群建立连接，或者在Meta数据需要被重复发送

## NetworkTimeouts结构

socket连接的配置

## Metadata结构

包含从Kafka请求Meta数据用到的配置

## Producer结构

包含生产者重试相关的选项

## Consumer结构

包含消费者重试相关的选项

# 变量与方法

## defaults变量

定义了默认的配置

## Load方法

Load方法解析orderer.yaml文件和环境变量，生成一个TopLevel

## completeInitialization方法

解析orderer的配置并初始化TopLevel参数

## translateCAs方法

返回将CA与configDir拼接
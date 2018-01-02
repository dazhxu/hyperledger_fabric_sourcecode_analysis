main
===

main.go文件作为orderer功能模块的入口，实现了orderer的命令及功能流程。

# 全局变量

## 设置

```golang
var logger = logging.MustGetLogger("orderer/main")

var (
	app = kingpin.New("orderer", "Hyperledger Fabric orderer node")

	start = app.Command("start", "Start the orderer node").Default()
	version = app.Command("version", "Show version information")
)
```

# 方法

## main方法

main方法是orderer命令的入口，对start命令和version命令进行了处理。

如果start命令，首先调用orderer/localconfig模块中的config.Load()方法把配置加载上去。

然后调用initializeLoggingLevel(conf)初始化日志级别；

调用initializeProfilingService服务初始化配置监听服务；

调用initializeGrpcServer(conf)初始化Grpc服务器；

调用initializeLocalMsp(conf)初始化本地MSP，然后调用common/localmsp的NewSigner方法创建签名器signer；

调用initializeMultiChainManager(conf, signer)创建多通道管理器manager；

调用server文件中的NewServer(manager, signer)创建服务器server。

调用protos.orderer的RegisterAtomicBroadcastServer(grocServer.Server(), server)将grpc服务器注册到server上。

最后调用grpcServer.Start()方法启动服务。

## initializeLoggingLevel方法

设置日志级别

```golang
flogging.InitBackend(flogging.SetFormat(conf.General.LogFormat), os.Stderr)
flogging.InitFromSpec(conf.General.LogLevel)
if conf.Kafka.Verbose {
	sarama.Logger = log.New(os.Stdout, "[sarama] ", log.Ldate | log.Lmicroseconds | log.Lshortfile)
}
```

## initializeProfilingService方法

如果开启了conf.General.Profile.Enabled，启动配置服务

```golang
if conf.General.Profile.Enabled {
	logger.Info("Starting Go pprof profiling service on:", conf.General.Profile.Address)
	logger.Panic("Go pprof service failed:", http.ListenAndServe(conf.General.Profile.Address, nil))
}
```

## initializeGrpcServer方法

初始化gprc服务器。

首先调用initializeSecureServerConfig初始化安全服务器配置secureConfig，调用net.Listen创建监听端口，然后调用core/comm模块的NewGRPCServerFromListen方法创建grpc服务器。

### initializeSecureServerConfig方法

首先创建comm.SecureServerConfig对象secureConfig。如果secureConfig.UseTLS被启用，创建secureConfig的其他选项。

从conf.General.TLS.Certificate读取证书文件，从conf.General.TLS.PrivateKey中读取私钥。从conf.GenralTLS.RootCAs中读取rootca证书，从conf.General.TLS.ClientRootCAs读取客户端证书。将这些设置到secureConfig的对应选项中。

## initializeLocalMsp方法

调用msp/mgmt模块中的mspmgmt.LoadLocalMsp方法加载本地MSP目录、BCCSP和MSPID。

## initializeMultiChainManager方法

初始化多通道管理器。

参数：

- conf *config.TopLevel
- signer crypto.LocalSigner

返回值：

- multichain.Manager

调用orderer/util文件中的createLedgerFactory方法创建账本工厂类lf。

如果lf.ChainIDs的长度为零，调用initializeBootstrapChannel方法创建初始通道。

创建名为consenters的映射，将创建"solo"和"kafka"类型的multichain.Consenter。

```golang
consenters := make(map[string]multichain.Consenter)
consenters["solo"] = solo.New()
consenters["kafka"] = kafka.New(conf.Kafka.TLS, conf.Kafka.Retry, conf.Kafka.Version)
```

最后，调用multichain.NewManagerImpl创建管理器

## initializeBootstapChannel方法

如果conf.General.GenesisMethod是provisional，调用provisional.New(genesisconfig.Load(conf.General.GenesisProfile)).GenesisBlock()方法创建
创世区块。

如果conf.General.GenesisMethod是file，调用file.New(conf.General.GenesisFile).GenesisBlock()创建创世区块。

然后调用lf.GetOrCreate(chainID)获取chainID，调用lf.GetOrCreate(chainID)创建systemchain，并在系统通道中添加genesisBlock
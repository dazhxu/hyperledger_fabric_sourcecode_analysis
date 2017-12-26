integration
===

integration模块用来启动一个gossip实例或leader election服务实例

# NewGossipComponent方法

创建一个gossip组件，并将其附加到给定的gRPC服务器上

参数：

- peerIdentity []byte
- endpoint string
- s *grpc.Server
- secAdv api.SecurityAdvisor
- cryptSvc api.MessageCryptoService
- idMapper identityMapper
- secureDialOpts api.PeerSecureDialOpts
- bootPeers ...string

返回值：

- gossip.Gossip
- error

首先调用newConfig(endpoint, externalEndpoint, bootPeers...)创建配置，然后调用gossip.NewGossipService方法创建gossip实例并返回

# newConfig方法

创建gossip.Config结构
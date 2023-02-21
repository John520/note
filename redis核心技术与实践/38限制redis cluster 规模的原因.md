## 通讯开销限制 redis cluster 规模
官方建议一个集群运行不超过1000个实例

## redis cluster 通信方式
 redis cluster 需要会保存每个slot 对应的是实例信息 ，redis cluster 采用gossip 协议通信
* 每个实例之间按照一定频率,从集群中随机挑选一些实例,发送Ping，交换彼此的状态信息及部分其他实例的信息，以及slot映射表
* 接收到Ping的实例会回复Pong,其消息内容和Ping消息一样

## 通信开销受那些影响
* 消息大小（12KB左右）。包括十分之一集群实例的自身状态信息和16384bitmap（表示每个实例对应的slot）
* 通信频率. 每秒随机5个实例，从中挑选最久未通信的实例发送Ping,同时每100ms检查最近应答Pong 
超过 `cluster-node-timeout/2`，立即向其发送Ping

调大`cluster-node-timeout`，可以减少实例间心跳占用的带宽

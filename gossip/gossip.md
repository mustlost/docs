# Gossip #

-> 在所有的Agent之间（包括服务器模式和普通模式）运行着Gossip协议。服务器节点和普通Agent都会加入这个Gossip集群，收发Gossip消息。每隔一段时间，每个节点都会随机选择几个节点发送Gossip消息，其他节点会再次随机安选择其他几个节点接力发送消息。这样一段时间过后，整个集群都能收到这条消息。

-> Gossip协议的最大的好处是，及时集群节点的数量增加，每个节点的负载也不会增加很多，几乎是恒定的。

-> Gossip算法又被称为反熵。Gossip天然具有分布式容错的优点。
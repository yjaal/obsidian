```toc
```


## 基本概念

Kafka 与其他 MQ 之间的区别就是，RabbitMQ（高级餐厅）支持复杂路由规则但吞吐有限；RocketMQ（大型餐馆）擅长处理复杂订单，支持事务消息、延迟消息，但架构复杂。而 kafka 不只是餐厅，而是食品供应链。

![](./img/002.png)


这里找了一张老版本的架构图，不过对于理解不影响。

可以看到一个 Topic 消息会分布在多个 Broker，这个 Broker 可以理解为一个物理节点。而一个 Broker 上面会存在多个逻辑分片，一个 Topic 会有多个分片，partition 会有多个副本，分布在不同的 broker 上面。

### Partition


#### Partition 是什么？

一个 **Topic（主题）** 在逻辑上是一个消息类别，但物理上它会被拆分成多个 **Partition**。
- 每个 Partition 是一个**有序的、不可变的、追加写的日志文件**（Commit Log）。
- 每条消息进入 Partition 时会被分配一个**单调递增的 Offset（偏移量）**。
- **Partition 内部有序，但 Partition 之间无序**。


#### 为什么需要 Partition？

1. 水平扩展（突破单机瓶颈）
- 一个 Partition 只能存在于一个 Broker 上。
- 如果 Topic 只有一个 Partition，那它只能落在一台机器上，磁盘、网络、CPU 都会成为瓶颈。
- 拆成多个 Partition 后，可以分散到多台 Broker，实现**存储和吞吐的水平扩展**。

2. 并行消费（提高消费能力）
- 在**同一个消费者组**内，**一个 Partition 同一时刻只能被一个消费者消费**。
- 所以 Partition 数决定了**消费者组内最大并行度**。
- 如果有 6 个 Partition，消费者组最多可以有 6 个消费者同时消费；多出来的消费者会空闲。
    

> **注意**：Partition 数不是越多越好。过多会导致：
> - Broker 元数据膨胀
> - 文件句柄增多
> - Leader 选举变慢
> - 端到端延迟增加

3. 顺序性保证
- Kafka 只保证**单个 Partition 内有序**。
- 如果业务要求某类消息严格有序，就把它们路由到同一个 Partition（比如用相同 Key）。

#### Partition 与消费者组的关系
- **同一消费者组内**：一个 Partition 只能被一个消费者消费。
- **不同消费者组之间**：互不影响，各自维护自己的 Offset。
- **Rebalance**：当消费者数量变化或 Partition 数变化时，会触发分区重新分配。



## 服务发现

Kafka 使用 KRaft 替换 zk。

Kafka 早期依赖 ZooKeeper [管理元数据](https://zhida.zhihu.com/search?content_id=262866110&content_type=Article&match_order=1&q=%E7%AE%A1%E7%90%86%E5%85%83%E6%95%B0%E6%8D%AE&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODkxOTkyODAsInEiOiLnrqHnkIblhYPmlbDmja4iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNjI4NjYxMTAsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.TNlrgcQJiEMhReZQ7g7FwF5UBVHL3illlm42pyjOQaQ&zhida_source=entity)，但这带来了根本性问题：

|问题|ZooKeeper 限制|Kafka 需求|
|---|---|---|
|数据模型|ZNode 树状结构|分区元数据扁平化|
|读写比例|读多写少|写频繁(分区变动)|
|数据规模|小规模元数据(<100 万节点)|大规模分区元数据(百万+)|
|一致性模型|强一致性|最终一致性即可|

**实际痛点**：

- ZooKeeper 的 ZNode 数量限制导致大规模集群问题
- ZooKeeper 的写性能瓶颈(约 1 万/秒)限制 Kafka 扩展
- 双系统运维复杂度翻倍，故障排查困难


## 数据写入

写入时可以指定分片，或者不指定分片。

指定分片时直接写入，若不指定，则根据传入的 key 来决定使用哪个分片，路由策略时
`hash(key) % 分片数量`。**每个 Partition 都有自己的 Leader 和 Follower**，而且 Leader 会尽量分散到不同 Broker，避免单点压力。在写入时只能选择 leader 分片。

注意：生产者会**定期向 Broker 拉取元数据**（Metadata），缓存在本地。元数据包含：

- 每个 Topic 有哪些 Partition
- 每个 Partition 的 Leader 在哪个 Broker
- 每个 Partition 有哪些 Replica、ISR 是谁























### 生产和消费工作流
如图：
![生产和消费工作流](https://www.processon.com/view/link/5f40612f1e085306e16e1f7a)


消费者组，从 topic 中消费消息，一个消费者组，只能消费一个 topic 上的一个分区的数据一次，但是一个分区上的数据可以被多个消费者组消费，

### 分区个数
topic 的分区数可以增加，但是不能减少，在计算分区个数的时候行之有效的方法是: topic 的吞吐量 / 消费者的吞吐量，如果无法知道吞吐量，将单个分区大小限制在 25 G 以内，一般来说能得到比较理想的效果。

### 生产者分区策略
- 分区原因
  1. 方便在集群中扩展，每个 Partition 可以通过调整以适应它所在的机器，一个 topic 又可以有多个 Partition 组成，因此整个集群就可以适应任意大小的数据
  2. 以 Partition 为单位读写，可以提高并发

- Producer 写入分区原则
一个 topic 可以分多个区，producer 可以向一个 topic 的指定分区写数据，producer 选取写入分区的原则如下：

  1. 指定了分区号，直接向指定的分区号写入
  2. 未指定分区号，但是消息是有 key 和 value 的，根据 key 计算 hash 值，然后和 topic 的可用分区总数取余
  3. 未指定分区，也没有 key 的情况下，第一次写入生成随机整数，之后每次在这个整数上递增，每次使用这个整数和 topic 的可用分区数取余

### 数据可靠性保证
- ack 机制

    为保证 producer 发送的数据，能可靠的发送给指定的 topic, topic 的每个 partition 收到 producer 发送的数据后，都需要向 producer 发送 ack, producer 收到 ack 后决定是否重发

- 合适发送 ack？

 每个 partition 是由多个副本组成的，在接收到 producer 的写入请求后，何时发送 ack？

 |编号|方案|优点|缺点|
|---|---|---|---|
|1|leader 写入数据完成后就发送  ack|延迟低|不能保证 follower 数据同步成功，重新选举 leader 后可能导致数据丢失|
|2|半数 follower 同步数据后发送 ack|延迟低|选举 leader 容忍 n 台节点故障，需要 2n + 1 个副本|
|3|全部副班同步完成后，发送 ack|选取 leader 时，容忍 n 台节点故障，只需要 n + 1 副本|延迟高|

 Kafka 读写速度很快，在 Kafka 集群中，网络对 Kafka 的延迟较小，Kafka 采用的是 3 号方案

- ISR 机制

 上面说到 Kafka 需要保证所有的 follower 同步完成数据才发送 ack，为了保证某些 follower 因为故障导致的同步数据耗时问题，Kafka 为每个 topic 维护了一个可用副本集合  in-sync replica set(ISR) ，ISR 维护的是副本中和 leader 通讯正常，从 leader 中同步数据正常的 follower, 如果 follower 长时间未从 leader 中同步数据，该 follower 会从 ISR 中移除，这个时间由 replica.lag.time.max.ms 参数指定的，默认 10000 ms，Leader 发生故障后，就会从 ISR 中选举新的 leader。

- ACK 应答机制

 对于某些不太重要的数据，每次都要求所有 follower 同步完再响应客户端很耗时，也许很多场景下我们可以容忍部分数据的丢失，来提高生产者和 broker 的交互效率，这时候生产者可以通过配置 acks 参数来指定 ACK 方式
  - acks
    - 0: 这中方式提供了一个最低的延迟，broker 一接收到的数据，还没有写入到磁盘就返回 ack，当 borker 故障时，可能导致写入数据丢失
    - 1:  producer 等待 broker 的 ack，partition 的 leader 写入数据成功后返回 ack，这种情况可能出现 follower 同步数据失败的情况
    - -1: producer 等待 broker 的 ack，需要等待 leader 和 follower 全部写入成功后的 ack， 这里的 follower 是 ISR 队列中的 follower。这种情况下，如果在 follower 同步完成后，broker 发送 ack 之前，leader 发生故障，那么会造成数据重复

- LEO 和 HW 机制

  kafka 数据的消费一致性和存储的可靠性，依赖的 LEO 和 HW 机制

 如图：
![kafka-HW&LEO]()

   LEO: 指的是每个副本的最大 offset
  HW: ISR 队列中副本共同的最大 offset， 也是消费者所能见到的最大 offset

  -  follower 故障

 follower 发生故障后，会被临时踢出 ISR, 待该 follower 恢复后，follower 会读取本地的磁盘记录上的 HW，并将 log 文件高于 HW 的部分截取掉，从 HW 向 leader 进行同步，等该 follower 的 LEO 大于等于该 Partion 的 HW，即 follower 追上 leader 后，就可以重新加入到 ISR

  - leader 故障
 
 leader 发生故障后，会从 ISR 中选取一个新的 leader，为保证多个副本之间的数据一致性，其余的 follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 leader 同步数据。

-  Exactly Once 

 Exactly Once 即精准一次性，指的是保证数据的不丢失，也不重复。kafka 使用 At least Once + 幂等性 来实现 Exactly Once, At least Once 保证的是数据不丢失，幂等性保证的是数据 Producer 不论向 Server 发送多少次重复的数据，Server 端只会持久化一条。只需要将 Producer 的 enable.idompotence 设置为 true, 即可启用幂等性。Broker 是通过判断  <PID, Partition,  SeqNumber> 来保证数据只会被持久化一次。PID 是 producer ID，Partition 是分区标识，Sequence Number 是消息的序列号，Kafka 通过这种方式来保证一条消息的唯一性，从而确保一条消息只会被持久化一次。
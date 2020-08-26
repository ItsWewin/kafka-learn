#### 消费方式
consumer 采用 pull 的方式从 broker 中读取数据。broker  push 的方式，数据发送速率由 broker 决定，很难满足不同处理能力消费者，pull 这种模式不足之处是，消费者需要循环的查询 broker，如果 kafka 没有数据，此时消费者就会陷入一个 pull 查询的循环中，但是每次拿到的都是空数据。针对这个问题，kafka 的消费者在消费数据时会传入一个时常参数 timeout，如果当前没有数据可消费，consumer 会等待一段时间后再返回，这段时间长度即 timeout

#### 分区策略
一个 consumer group 中有多个 consumer，一个 topic 有多个 partition，所以就涉及到 partition 的分配问题，即确定那个 partition  由哪个 consumer 来消费的问题。

Kafka 有两种分配策略，一是 roundRobin, 一是 Range.

- RoundRobin
 会将当前消费者组订阅的所有 topic 的所有 partion 进行重新排序，然后循环分配给消费者组里的消费者，使用这种方式要求一个消费者组中的消费者订阅相通的 topic. 负责会导致某些消费者消费了自己没有订阅的消息。

- Range
这种方式是按照单个 topic 的 partition 进行 range，将 partition 循环分配给消费者组中订阅了该 topic 的消费者，没有订阅的不会分配

只要消费者组中的消费者数量发生变化时，就会触发重新分配，有种场景，比如 topic-a 有两个分区 p1 和 p2，已经分配给消费者组 topic-group-a 中的 c1、c2 两个消费者，此时如果增加一个消费者 c3，这种场景下也是要重新分配的。

*kafka 默认是使用的 range 这种方式*

#### offset 的维护
consumer 在消费过程中可能会出现故障，需要在 consumer 恢复后，需要从故障前的位置继续消费，所以 consumer 需要实时记录自己消费到哪个 offset 了，以便故障恢复后继续消费。kafka 0.9 之前，consumer 默认将 offset 保存在 zookeeper 中，从 0.9 版本开始，consumer 默认将 offset 保存在 kafka 一个内置的 topic 中，该 topic 为 __consumer_offsets, 消费者组 + topic + partition 决定唯一的一个消费者的 offset
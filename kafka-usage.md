# Kafka 基本使用 

### 主题操作
- 创建一个主题

 ```
$ ./bin/kafka-topics.sh --create --zookeeper 192.168.56.23：2181 --topic first --partitions 2 --replication-factor 2
Created topic first.
 ```
  - 参数说明：
    - --partitions: 指定分区数
    - --replication-factor： 指定副本数，副本数不能对于 broker 
    - --topic：指定 topic 名称
  - 如果参数 replication-factor 大于 broker 数（我使用 kafka 集群的 broker 数是 3）
   ```
    $ ./bin/kafka-topics.sh --create --zookeeper 192.168.56.23:2181 --topic third --partitions 1 --replication-factor 5
Error while executing topic command : Replication factor: 5 larger than available brokers: 3.
[2020-08-20 14:33:17,455] ERROR org.apache.kafka.common.errors.InvalidReplicationFactorException: Replication factor: 5 larger than available brokers: 3.
 (kafka.admin.TopicCommand$)
   ```
    
 这里指定分区数 5 大于 brokers 总数 3，创建失败

- 查看主题

 ```
$ ./bin/kafka-topics.sh --list --zookeeper 192.168.56.23:2181
first
 ```
- 查看数据

 此时到 kafka 的数据目录：
 ```
$ cd /opt/release/kafka_2.13-2.6.0/data
$ ls
cleaner-offset-checkpoint  first-1			meta.properties			  replication-offset-checkpoint
first-0			   log-start-offset-checkpoint	recovery-point-offset-checkpoint
 ```
可以看到有 first-0 和 first-1 这就是刚才创建的 topic, 0 和 1 代表的是两个分区


- 删除 topic

 ```
$ cd /opt/release/kafka_2.13-2.6.0
$ ./bin/kafka-topics.sh --delete --zookeeper 192.168.56.23:2181 --topic first
Topic first is marked for deletion.
Note: This will have no impact if delete.topic.enable is not set to true.  // config/server.properties 文件中 delete.topic.enable=true 会直接删除
 ```

- 产看 topic 详情

 ```
$ ./bin/kafka-topics.sh --create --zookeeper 192.168.56.23:2181 --topic sencode --partitions 2 --replication-factor 3
$ ./bin/kafka-topics.sh --describe --topic sencode --zookeeper 192.168.56.23:2181
Topic: sencode	PartitionCount: 2	ReplicationFactor: 3	Configs:
	Topic: sencode	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
	Topic: sencode	Partition: 1	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
 ```

### 生产和消费消息
- 生产消息

 在任意一台 kafka 服务器上，执行下面的命令
 ```
 $ $ ./bin/kafka-console-producer.sh --topic book --broker-list 192.168.56.22:9092 # 192.168.56.22 也可以换成其他 kafka 节点的 ip
 ```

- 消费消息

 在其他一台或者多台 kafka 节点上，执行下面的命令

 ```
$ ./bin/kafka-console-consumer.sh --topic book --bootstrap-server 192.168.56.22:9092
 ```
 消费消息也可以加上 `--from-beginning` 参数，可以消费生产者生产的历史消息(kafka 的历史消息有最大保存时间，默认是 7 天)

 ```
 $ ./bin/kafka-console-consumer.sh --topic book --bootstrap-server 192.168.56.22:9092 --from-beginning
 ```
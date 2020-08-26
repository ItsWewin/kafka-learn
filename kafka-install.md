# 简介
### 是什么？

 Kafka 是一个分布式的基于发布 / 订阅模式的消息队列，主要用于大数据实时处理领域。

### 什么用？
  1. 解耦：不同系统模块之间可以独立的扩展和修改，只要保证接口一致性，解耦后也有利于系统的恢复，可以保证消息不因为处理消息的进程挂掉而丢失
  2. 缓冲：有助于控制和优化数据经过系统的速度，解决生产者和消费者的处理速度不一致的情况
  3. 灵活性和峰值处理能力（削峰）：保证访问量剧增的情况下，系统仍然可用，不用为短时间的突发流量准备随时待命的服务资源
  4. 异步通讯：不用实时处理的消息放入消息队列，消费者使用的时候，随时来获取消费

### 发布订阅模式

消息队列有两种模式 `点对点模式`  和 `发布订阅模式`， Kafka 使用的是 `发布订阅模式`， 两种模式有什么区别？

1. 点对点模式 (一个消息只能被消费一次，消费者主动拉取数据，消息收到后会清除消息)

 ![点对点模式]()

 生产者生产消息发送到队列中，消费者从队列中取出消息并消费，消息消费后，队列中不再存储，队列支持多个消费者，但是对于一条消息，只会被一个消费者消费

2. 发布订阅模式

 ![发布订阅模式]()

  生产者发布消息到 topic 中，同时有多个消费者订阅消息，发布者发布的消息可以被所有的订阅着消费。

### Kafka 基础架构
![Kafka基本架构]()

- Producer: 消息生产者，是向 Kafka broker 发布消息的客户端
- Consumer： 消费者，向 Kafka broker 获取消息的客户端
- Consumer Group: 消费者组，由一个或者多个 consumer 组成，消费者组里的消费者负责消费不同分区的数据，一个分区只能由一个消费者组内的消费者消费，消费者组之间互不影响。所有的消费者都属于某个消费者组，消费者组是一个逻辑上的订阅着
- Broker： 一台 Kafka 服务器就是一个 broker，一个集群由多个 broker 组成，一个 broker 有多个 topic
- Topic: 消息的分类，可以理解为一个队列，生产者和消费者面向的是 topic
- Partition: 一个 potic 可以分为多个 partition，每个 partion 是一个有序的队列，可以实现一个 topic 分不到不用的 broker 上
- Replication：副本，可以理解为 partition 的一个备份，保证不因某个节点的故障而都是数据，一般一个 partition 包含一个 leader 和多个 follower
- Leader： 每个分区多个 Replication 的 “主”，生产者发送、和消费者获取数据面向的是 leader
- Follower: 每个分区多个 Replication 中的 “从”， 实时的从 leader 中同步数据，保持和 leader 数据的同步，leader 发证故障的时候，某个 follower 会成为新的 leader


# 部署
### 资源准备
1. Kafka 集群规划

 |server name|ip|remark|
|---|---|---|
|server1|192.168.56.21|Kafka 节点1|
|server2|192.168.56.22|Kafka 节点2|
|server3|192.168.56.23|Kafka 节点3|

 

2. zookeeper 集群规划

 |server name|ip|remark|
|---|---|---|
|server1|192.168.56.21|zookeeper 节点1|
|server2|192.168.56.22|zookeeper 节点2|
|server3|192.168.56.23|zookeeper 节点3|


3. 软件版本说明

 |软件|版本|下载|
|---|---|---|
|ubuntu|18.04|[ubuntu 官网](https://ubuntu.com/download/server)|
|kafka|2.6.0|[下载地址](https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.6.0/kafka_2.13-2.6.0.tgz)|
|zookeeper|3.4.9|[下载地址](https://zookeeper.apache.org/releases.html#download)|

*Java 是必备的*

### Kafka 安装
1. server1 （192.168.56.21）
 - 下载并解压 kafka
 ```
 $ sudo mkdir /opt/release
 $ sudo mkdir ~/downloads
 $ cd ~/downloads
 $ wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.6.0/kafka_2.13-2.6.0.tgz  # 这里的 2.13 是 Scala 版本
 $ sudo tar -zvxf kafka_2.13-2.6.0.tgz -C /opt/release/
 ```
- 修改配置
 ```
$ cd /opt/release/kafka_2.13-2.6.0
$ mkdir data
$ sudo vim config/server.properties # 修改如下两行
`
log.dirs=/opt/release/kafka_2.13-2.6.0/data # 数据位置，默认 Kafka 数据和日记都放在这个目录下，指定后，数据会到指定的目录中，日志放在 Kafka 根目录下的 logs 下，这样数据和日志就可以分开，logs 文件会自动创建 
zookeeper.connect=192.168.56.21:2181,192.168.56.22:2181,192.168.56.23:2181 # zookeeper 节点
broker.id=0 # 修改 Kafka 节点 id
listeners=PLAINTEXT://192.168.56.21:9092 # Kafka 监听的节点
`
修改后保存退出
 ```

2. server2 （192.168.56.22）
 - 下载并解压 Kafka (建议从 server0 scp)
 ```
 $ sudo mkdir /opt/release
 $ sudo mkdir ~/downloads
 $ cd ~/downloads
 $ wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.6.0/kafka_2.13-2.6.0.tgz
 $ sudo tar -zvxf kafka_2.13-2.6.0.tgz -C /opt/release/
 ```
- 修改配置
 ```
$ cd /opt/release/kafka_2.13-2.6.0
$ mkdir data
$ sudo vim config/server.properties # 修改如下两行
`
log.dirs=/opt/release/kafka_2.13-2.6.0/data # 数据和日志位置，Kafka 数据文件也是 *.log 文件
zookeeper.connect=192.168.56.21:2181,192.168.56.22:2181,192.168.56.22:2181 # zookeeper 节点
broker.id=1 # 修改 Kafka 节点 id
listeners=PLAINTEXT://192.168.56.22:9092 # Kafka 监听的节点
`
修改后保存退出
 ```

3. server3 （192.168.56.23）
 - 下载并解压 Kafka （建议从 server0 scp）
 ```
 $ sudo mkdir /opt/release
 $ sudo mkdir ~/downloads
 $ cd ~/downloads
 $ wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.6.0/kafka_2.13-2.6.0.tgz
 $ sudo tar -zvxf kafka_2.13-2.6.0.tgz -C /opt/release/
 ```
- 修改配置
 ```
$ cd /opt/release/kafka_2.13-2.6.0
$ mkdir data
$ sudo vim config/server.properties # 修改如下两行
`
log.dirs=/opt/release/kafka_2.13-2.6.0/data # 数据和日志位置，Kafka 数据文件也是 *.log 文件
zookeeper.connect=192.168.56.21:2181,192.168.56.22:2181,192.168.56.23:2181 # zookeeper 节点
broker.id=2 # 修改 Kafka 节点 id
listeners=PLAINTEXT://192.168.56.22:9092 # Kafka 监听的节点
`
修改后保存退出
 ```

4. 启动服务

 在 server1, server2, server3 上启动 Kafka 服务
 ```
$ cd /opt/release/kafka_2.13-2.6.0
$ sudo ./bin/kafka-server-start.sh -daemon config/server.properties
 ```
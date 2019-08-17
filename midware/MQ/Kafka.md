1.**基本概念与原理介绍：**
- producer端发送消息已topic进行归类，构造ProducerRecord时需要指定topic参数，每个topic又有很多分区(可配置)，每个分区本质是可追加的log文件，因此kafka保证消息在单分区的顺序性，但不能保证topic的顺序性。众多分区有一个首领分区，负责读写，其他事follower分区，负责同步首领分区的数据，不对外读写操作。
- kafka是基于pull的方式，consumer端去拉broker端的消息。
- consumer端维护读消息的偏移量，不由broker维护，由zookeeper的元数据维护。
- 一个简单的kafka服务器是一个broker.
- 首领副本  
    - 每个分区都有一个首领副本。为了保证一致性，所有生产者请求和消费者请求都会经过
这个副本。
- 跟随者副本
    - 首领以外的副本都是跟随者副本。跟随者副本不处理来自客户端的请求，它们唯一的任
务就是从首领那里复制消息，保持与首领一致的状态。如果首领发生崩渍，其中的一个
跟随者会被提升为新首领。

2.**kafka启动：**
- 先启动自带zk：
    - ./zookeeper-server-start.sh -daemon ../config/zookeeper.properties
- 再启动kafka: 
    - ./kafka-server-start.sh -daemon ../config/server.properties(启动端口为9092,此时broker监听的端口是9092)
- 可以再启动zk-shell： 
    -  ./zookeeper-shell.sh localhost:2181
- 在kafka命令行创建topic: 
    - ./kafka-topics.sh --create --topic java_topic --zookeeper localhost:2181 --partitions 1 --replication-factor 1(副本为一，表示集群只保存一份数据)
- 命令行创建producer客户端:
     -./kafka-console-producer.sh --topic test --broker-list 192.168.1.102:9092
- 命令行创建consumer客户端：
    - ./kafka-console-consumer.sh --topic test --bootstrap-serve 192.168.1.102:9092
- 命令行创建consumer客户端： 
     -./kafka-console-consumer.sh --topic test --bootstrap-serve 192.168.1.102:9092 --from-beginning 加上--from-beginning可以从头开始pull生产者端的消息。

**在虚拟机启动kafka的问题解决：**
- ```注意：```把配置文件中的所有localhost或者默认主机名都换成虚拟机的ip地址，这样可以减少很多windows与虚拟机通信的问题。
- **kafka启动没一会就挂了？**
    - 因为虚拟机是dhcp，所以IP会改变，因此需要去修改server.properties文件的IP相关。
    - 如果上述不行：把kafka的server.properties中zookeeper.connection.timeout.ms=60000设置为60s，同时需要，如果使用的不是kafka自带的zk,而是另自安装的zk,那么需要更改zoo.cfg中的tickTime=20000时间设置为20s，表示zk的服务器心跳时间。
    - 

HDD传统机械硬盘。SSD固态硬盘。

3.**kafka的配置介绍：**
- log.dirs: 
    - Kafka 把所有消息都保存在磁盘上，存放这些日志片段的目录是通过 log.dirs指定的.


4.**分析kafka源码知道**：
- 在```ProducerConfig```里面有很多producer端想要的配置信息，比如partitioner,interceptor
- 在```ConsumerConfig```里面有很多consumer端想要的配置信息

5.**Java的logback日志配置：**
- https://www.cnblogs.com/sky230/p/6420208.html

6.**kafka的原理：**
- broker 会在它所监听的每一个端口上运行一个 Accepto 「线程，这个钱程会创建一个连接，并把它交给 processor线程去处理。 Processor线程（也被叫作“网络线程”）的数量是可配置的。网络线程负责从客户端获取请求悄息，把它们放进请求队列，然后从晌应队列获取响应消息，把它们发送给客户端。请求消息被放到请求队列后，IO线程会负责处理它们。

7.**kafka server.properties介绍**
```shell
# broker 的编号，如果集群中有多个broker ，则每个broker 的编号需要设置的不同
broker.id=0
# broker 对外提供的服务入口地址
listeners= PLAINTEXT://localhost:9092
# 存放消息日志文件的地址
log.dirs=/tmp/kafka-logs
# Kafka 所需的ZooKeeper 集群地址，为了方便演示，我们假设Kafka 和ZooKeeper 都安装在本机
zookeeper.connect=localhost:2181/kafka
```

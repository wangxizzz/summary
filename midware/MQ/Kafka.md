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
- 分区中 的所有副本统称为 AR ( Assigned Replicas ） 。 所有与 leader 副本保持一定程度同步
的副本（包括 leader 副本在内〕组成 ISR On-Sync Replicas ) , ISR 集合是 AR 集合中 的一个子
集 。 消息会先发送到 leader 副本，然后 follower 副本才能从 leader 副本中拉取消息进行同步，
同步期间内 follower 副本相对于 leader 副本而言会有一定程度的滞后 。 前面所说的“一定程度
的同步”是指可忍受的滞后范围，这个范围可以通过参数进行配置 。 与 leader 副本同步滞后过
多的副本（不包括 leader 副本）组成 OSR ( Out-of-Sync Replicas ），由此可见， AR=ISR+OSR 。
在正常情况下， 所有的 follower 副本都应该与 leader 副本保持一定程度 的同步，即 AR=ISR,
OSR 集合为空。

2.**kafka启动,版本为2.1.0，zk版本3.1.14：**
- 先启动自带zk：
    - ./zookeeper-server-start.sh -daemon ../config/zookeeper.properties
- 再启动kafka: 
    - ./kafka-server-start.sh -daemon ../config/server.properties(启动端口为9092,此时broker监听的端口是9092)
- 可以再启动zk-shell： 
    -  ./zookeeper-shell.sh localhost:2181
- 在kafka命令行创建topic: 
    - ./kafka-topics.sh --create --topic java_topic --zookeeper localhost:2181 --partitions 1 --replication-factor 1(副本为一，表示集群只保存一份数据)
- 命令行创建producer客户端:
     -./kafka-console-producer.sh --topic java_topic --broker-list 192.168.1.102:9092
- 命令行创建consumer客户端：
    - ./kafka-console-consumer.sh --topic java_topic --bootstrap-server 192.168.1.102:9092
- 命令行创建consumer客户端： 
     -./kafka-console-consumer.sh --topic java_topic --bootstrap-server 192.168.1.102:9092 --from-beginning 加上--from-beginning可以从头开始pull生产者端的消息。
- 查看有哪些topic:
    - ./kafka-topics.sh --zookeeper localhost:2181 -list
- 查看topic的描述信息：
    - ./kafka-topics.sh --zookeeper localhost:2181 --describe --topic java_topic
- 删除主题：
    - 必须先开启配置：delete.topic.enable=true
    - ./kafka-topics.sh --zookeeper localhost:2181 --delete --topic java_topic
- 



**在虚拟机启动kafka的问题解决：**
- ```注意：```把配置文件中的所有localhost或者默认主机名都换成虚拟机的ip地址，这样可以减少很多windows与虚拟机通信的问题。
- **kafka启动没一会就挂了？**
    - 因为虚拟机是dhcp，所以IP会改变，因此需要去修改server.properties文件的IP相关。
    - 如果上述不行：把kafka的server.properties中zookeeper.connection.timeout.ms=60000设置为60s，同时需要，如果使用的不是kafka自带的zk,而是另自安装的zk,那么需要更改zoo.cfg中的tickTime=20000时间设置为20s，表示zk的服务器心跳时间。
    - 

HDD传统机械硬盘。SSD固态硬盘。

3.**kafka常见配置介绍：**
- log.dirs: 
    - Kafka 把所有消息都保存在磁盘上，存放这些日志片段的目录是通过 log.dirs指定的.
- zk的chroot: 
    - chroot是一个zk的namespace
- **生产端配置：**
    - buffer.memory:
        - RecordAccumulator的缓存大小，producer端批量发送的缓冲区
    - acks:
        - 这个参数用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消
息是成功写入的。
    - 
- **消费端配置：**
    - enable.auto.commit:
        - 自动提交偏移量，默认为true. 这个逻辑是在poll()方法完成
    - auto.offset.reset:
        - 在 Kafka 中 每当消费者查找不到所记录的消费位移 时， 默认是latest,表示最近提交的偏移量的下一个开始。还有值为earliest,表示从该分区0开始消费。
        - 在位移越界也会触发参数运行
        - 如果配置为none，那么在发生上述两种情况就会抛出ConfigException
    - 

4.**分析kafka源码知道**：
- 在```ProducerConfig```里面有很多producer端想要的配置信息，比如partitioner,interceptor
- 在```ConsumerConfig```里面有很多consumer端想要的配置信息
- 使用seek()可以指定位移消费。

5.**Java的logback日志配置：**
- https://www.cnblogs.com/sky230/p/6420208.html

6.**kafka的原理：**
- broker 会在它所监听的每一个端口上运行一个 Accepto 「线程，这个钱程会创建一个连接，并把它交给 processor线程去处理。 Processor线程（也被叫作“网络线程”）的数量是可配置的。网络线程负责从客户端获取请求悄息，把它们放进请求队列，然后从晌应队列获取响应消息，把它们发送给客户端。请求消息被放到请求队列后，IO线程会负责处理它们。
- KafkaProducer 是线程安全的，可以在多个线程中共享单个KafkaProducer 实例，也可以将
KafkaProducer 实例进行池化来供其他线程调用。
- **可重试异常与不可重试异常：**
    - KafkaProducer 中一般会发生两种类型的异常： 可重试的异常和不可重试的异常。常见的可
重试异常有： NetworkException 、LeaderNotAvailableException 、UnknownTopicOrPartitionException 、NotEnoughReplicasException 、NotCoordinatorException 等。比如NetworkException 表示网络异常，这个有可能是由于网络瞬时故障而导致的异常，可以通过重试解决；又比如LeaderNotAvailableException 表示分区的leader 副本不可用，这个异常通常发生在leader 副本下
线而新的leader 副本选举完成之前，重试之后可以重新恢复。不可重试的异常，比如1.4 节中
提及的RecordTooLargeException 异常，暗示了所发送的消息太大， KafkaProducer 对此不会进行
任何重试，直接抛出异常。
- **kafka的偏移量：**
    - 为了防止broker重启，因此consumer提交的偏移量不能保存在内存中，需要持久化到磁盘.偏移量存储在kafka内部主题的_consumer_offset中。
    - 消费者需要提交的offset值为这次消费的最大offset + 1.

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

8.**重要的原理与概念:**
- 对于偏移量这种非常重要的东西，kafka的处理是持久化，而不是放到内存中，否则的机器一重启就没了。
- 消费者位移保存在内部主题_consumer_offsets中。客户端提交的位移值，是下一次消费的值。假如这一次消费的X，那么偏移量的提交为X+1.
- 在再均衡期间，消费者是不会读取消息的，也就是说对外不可用。
- KatkaProducer 是线程安全 的，然而 KafkaConsumer 却是非线程安全 的 。 KafkaConsumer 中
定义了 一个 acquire（）方法，用来检测当前是否只有一个线程在操作，若有其他线程正在操作则
会抛出 ConcurrentModifcationException 异常。

9.**顺序读写与随机读写的性能差异：**  
顺序读写和随机读写对于机械硬盘来说为什么性能差异巨大？
顺序读写=读取一个大文件  
随机读写=读取多个小文件  
顺序读写比随机读写快的原因  
（1）随机读写的磁头寻道时间大大增加，因为磁头需要不停的移动，而顺序读写就不用频繁的移动。  
（2）顺序读写，磁盘会预读，预读即在读取的起始地址连续读取多个页面  
（现在不需要的页面也读取了，这样以后用时就不用再读取，当一个页面用到时，大多数情况下，它周围的页面也会被用到）    
而随机读写，因为数据没有在一起，将预读浪费掉了。  
（3）另一个原因是文件系统的overhead。  
读写一个文件之前，得一层层目录找到这个文件，以及做一堆属性、权限之类的检查。  
写新文件时还要加上寻找磁盘可用空间的耗时。  
对于小文件，这些时间消耗的占比就非常大了。    
（4）**参照文章：https://tech.meituan.com/2017/05/19/about-desk-io.html**

10.
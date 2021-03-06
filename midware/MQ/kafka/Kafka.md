# 基本概念与原理介绍
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

# kafka常见配置介绍：
## broker端：
- log.dirs: 
    - Kafka 把所有消息都保存在磁盘上，存放这些日志片段的目录是通过 log.dirs指定的.
- zk的chroot: 
    - chroot是一个zk的namespace
## 生产端重要参数配置：
- buffer.memory:
    - RecordAccumulator的缓存大小，producer端批量发送的缓冲区。Kafka的客户端发送数据到服务器，不是来一条就发一条，而是经过缓冲的，也就是说，通过KafkaProducer发送出去的消息都是先进入到客户端本地的内存缓冲里，然后把很多消息收集成一个一个的Batch，再发送到Broker上去的，这样性能才可能高。```此参数是以batch为维度进行累加。``` buffer.memory的本质就是用来约束KafkaProducer能够使用的内存缓冲的大小的，默认值32MB。如果buffer.memory设置的太小，可能导致的问题是：消息快速的写入内存缓冲里，但Sender线程来不及把Request发送到Kafka服务器，会造成内存缓冲很快就被写满。而一旦被写满，就会阻塞用户线程，不让继续往Kafka写消息了。
- batch.size
    - 每个Batch要存放batch.size大小的数据后，才可以发送出去。比如说batch.size默认值是16KB，那么里面凑够16KB的数据才会发送。```以ProducerRecord为维度进行累加。```理论上来说，提升batch.size的大小，可以允许更多的数据缓冲在里面，那么一次Request发送出去的数据量就更多了，这样吞吐量可能会有所提升。但是batch.size也不能过大，要是数据老是缓冲在Batch里迟迟不发送出去，那么发送消息的延迟就会很高。一般可以尝试把这个参数调节大些，利用生产环境发消息负载测试一下。
- linger.ms
    - 一个Batch被创建之后，最多过多久，不管这个Batch有没有写满，都必须发送出去了。比如说batch.size是16KB，但是现在某个低峰时间段，发送消息量很小。这会导致可能Batch被创建之后，有消息进来，但是迟迟无法凑够16KB，难道此时就一直等着吗？当然不是，假设设置“linger.ms”是50ms，那么只要这个Batch从创建开始到现在已经过了50ms了，哪怕他还没满16KB，也会被发送出去。 
　　所以“linger.ms”决定了消息一旦写入一个Batch，最多等待这么多时间，他一定会跟着Batch一起发送出去。 
　　linger.ms配合batch.size一起来设置，可避免一个Batch迟迟凑不满，导致消息一直积压在内存里发送不出去的情况。
- max.request.size
    - 决定了每次发送给Kafka服务器请求消息的最大大小。如果发送的消息都是大报文消息，每条消息都是数据较大，例如一条消息可能要20KB。此时batch.size需要调大些，比如设置512KB，buffer.memory也需要调大些，比如设置128MB。 只有这样，才能在大消息的场景下，还能使用Batch打包多条消息的机制。此时“batch.size”也得同步增加。
- retries和retries.backoff.ms
    - 重试机制，也就是如果一个请求失败了可以重试几次，每次重试的间隔是多少毫秒，根据业务场景需要设置。
- acks:
    - "0"： Producer 往集群发送数据不需要等到集群的返回，不确保消息发送成功。安全性最低但是效率最高。
    - "1": Producer 往集群发送数据只要 Leader 应答就可以发送下一条，只确保 Leader 接收成功。
    - "-1"或"all" : Producer 往集群发送数据需要所有的```ISR Follower ```都完成从 Leader 的同步才会发送下一条，确保 Leader 发送成功和所有的副本都成功接收。安全性最高，但是效率最低。
- 
## 消费端重要参数配置：
- enable.auto.commit:
    - 自动提交偏移量，默认为true. 这个逻辑是在poll()方法完成
- auto.offset.reset:
    - 以下是经过真实场景测试的
    - **重启之前挂掉的consumer, 意思是重启后consumer group不变**
        - earliest：当服务器上toipc的该consumer group该分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费。
        - latest： 当toipc的consumer group该分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据
        - 如果配置为none，那么在发生上述两种情况就会抛出ConfigException，默认为latest 
    - **如果新加入一个consumer group或者重启后consumer group变了**
        - earliest：从该topic的offset=0开始消费
        - latest: 从该topic producer端产生的最新的offset开始消费，意思表示中间那段消息就不管了

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

# kafka消息发送流程
消息在真正发往Kafka之前，有可能需要经历拦截器（Interceptor）、序列化器（Serializer）和分区器（Partitioner）等一系列的作用，那么在此之后又会发生什么呢？

<img src="../../../imgs/kafka-producer.png" height=600px width=600px>

整个生产者客户端由两个线程协调运行，这两个线程分别为主线程和Sender线程（发送线程）。在主线程中由KafkaProducer创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息累加器（RecordAccumulator，也称为消息收集器）中。Sender 线程负责从RecordAccumulator中获取消息并将其发送到Kafka中。

RecordAccumulator 主要用来缓存消息以便 Sender 线程可以批量发送，进而减少网络传输的资源消耗以提升性能。RecordAccumulator 缓存的大小可以通过生产者客户端参数buffer.memory 配置，默认值为 33554432B，即 32MB。如果生产者发送消息的速度超过发送到服务器的速度，则会导致生产者空间不足，这个时候KafkaProducer的send（）方法调用要么被阻塞，要么抛出异常，这个取决于参数max.block.ms的配置，此参数的默认值为60000，即60秒。  

主线程中发送过来的消息都会被追加到RecordAccumulator的某个双端队列（Deque）中，在 RecordAccumulator 的内部为每个分区都维护了一个双端队列，队列中的内容就是ProducerBatch，即 Deque＜ProducerBatch＞。消息写入缓存时，追加到双端队列的尾部；Sender读取消息时，从双端队列的头部读取。注意ProducerBatch不是ProducerRecord，ProducerBatch中可以包含一至多个 ProducerRecord。通俗地说，ProducerRecord 是生产者中创建的消息，而ProducerBatch是指一个消息批次，ProducerRecord会被包含在ProducerBatch中，这样可以使字节的使用更加紧凑。与此同时，将较小的ProducerRecord拼凑成一个较大的ProducerBatch，也可以减少网络请求的次数以提升整体的吞吐量。如果生产者客户端需要向很多分区发送消息，则可以将buffer.memory参数适当调大以增加整体的吞吐量。  

消息在网络上都是以字节（Byte）的形式传输的，在发送之前需要创建一块内存区域来保存对应的消息。在Kafka生产者客户端中，通过java.io.ByteBuffer实现消息内存的创建和释放。不过频繁的创建和释放是比较耗费资源的，```在RecordAccumulator的内部还有一个BufferPool，它主要用来实现ByteBuffer的复用，以实现缓存的高效利用。```不过BufferPool只针对特定大小的ByteBuffer进行管理，而其他大小的ByteBuffer不会缓存进BufferPool中，这个特定的大小由batch.size参数来指定，默认值为16384B，即16KB。我们可以适当地调大batch.size参数以便多缓存一些消息。

ProducerBatch的大小和batch.size参数也有着密切的关系。当一条消息（ProducerRecord）流入RecordAccumulator时，会先寻找与消息分区所对应的双端队列（如果没有则新建），再从这个双端队列的尾部获取一个 ProducerBatch（如果没有则新建），查看 ProducerBatch 中是否还可以写入这个ProducerRecord，如果可以则写入，如果不可以则需要创建一个新的ProducerBatch。在新建ProducerBatch时评估这条消息的大小是否超过batch.size参数的大小，如果不超过，那么就以batch.size 参数的大小来创建 ProducerBatch，这样在使用完这段内存区域之后，可以通过BufferPool 的管理来进行复用；如果超过，那么就以评估的大小来创建ProducerBatch，这段内存区域不会被复用。

Sender 从 RecordAccumulator 中获取缓存的消息之后，会进一步将原本＜分区，Deque＜ProducerBatch＞＞的保存形式转变成＜Node，List＜ ProducerBatch＞的形式，其中Node表示Kafka集群的broker节点。对于网络连接来说，生产者客户端是与具体的broker节点建立的连接，也就是向具体的 broker 节点发送消息，而并不关心消息属于哪一个分区；而对于KafkaProducer的应用逻辑而言，我们只关注向哪个分区中发送哪些消息，所以在这里需要做一个应用逻辑层面到网络I/O层面的转换。

在转换成＜Node，List＜ProducerBatch＞＞的形式之后，Sender 还会进一步封装成＜Node，Request＞的形式，这样就可以将Request请求发往各个Node了，这里的Request是指Kafka的各种协议请求，对于消息发送而言就是指具体的ProduceRequest

# kafka的日志存储
## 日志文件
- 每个日志分段包含3个文件，后缀分别为：.log , .index, .timeindex文件
- .index结尾的文件为偏移量索引文件，每个索引项占用8个字节，分为两个部分。
    - （1）relativeOffset：相对偏移量，表示消息相对于baseOffset的偏移量，占用4 个字节，当前索引文件的文件名即为baseOffset的值。
    - （2）position：物理地址，也就是消息在日志分段文件(.log结尾的文件)中对应的物理位置，占用4个字节。
    - 从Log对象利用跳表存储logSegment，每个logSegment里的relativeOffset单调递增，因此查找offset时，先查跳表，再二分。
- 时间戳索引项占用12个字节，分为两个部分
    - timestamp：当前日志分段最大的时间戳。
    - relativeOffset：时间戳所对应的消息的相对偏移量。
- 时间戳索引文件中包含若干时间戳索引项，每个追加的时间戳索引项中的 timestamp 必须大于之前追加的索引项的timestamp，否则不予追加。
- Kafka 将消息存储在磁盘中，为了控制磁盘占用空间的不断增加就需要对消息做一定的清理操作。Kafka 中每一个分区副本都对应一个 Log，而 Log 又可以分为多个日志分段，这样也便于日志的清理操作。Kafka提供了两种日志清理策略：
    - （1）日志删除（Log Retention）：按照一定的保留策略直接删除不符合条件的日志分段
    - （2）日志压缩（Log Compaction）：针对每个消息的key进行整合，对于有相同key的不同value值，只保留最后一个版本(类似于redis aof重写机制)。
## kafka日志存储 & 高性能原因：
## 顺序追加：
- 有关测试结果表明，一个由6块7200r/min的RAID-5阵列组成的磁盘簇的线性（顺序）写入速度可以达到600MB/s，而随机写入速度只有100KB/s，两者性能相差6000倍。操作系统可以针对线性读写做深层次的优化，比如预读（read-ahead，提前将一个比较大的磁盘块读入内存）和后写（write-behind，将很多小的逻辑写操作合并起来组成一个大的物理写操作）技术。顺序写盘的速度不仅比随机写盘的速度快，而且也比随机写内存的速度快。
- Kafka 在设计时采用了文件追加的方式来写入消息，即只能在日志文件的尾部追加新的消息，并且也不允许修改已写入的消息，这种方式属于典型的顺序写盘的操作，所以就算 Kafka使用磁盘作为存储介质，它所能承载的吞吐量也不容小觑。
## 页缓存 
- 首先，Kafka重度依赖底层操作系统提供的PageCache功能。当上层有写操作时，操作系统只是将数据写入PageCache，同时标记Page属性为Dirty。当读操作发生时，先从PageCache中查找，如果发生缺页才进行磁盘调度，最终返回需要的数据。实际上PageCache是把尽可能多的空闲内存都当做了磁盘缓存来使用。同时如果有其他进程申请内存，回收PageCache的代价又很小，所以现代的OS都支持PageCache。使用PageCache功能同时可以避免在JVM内部缓存数据，JVM为我们提供了强大的GC能力，同时也引入了一些问题不适用于Kafka的设计。
- Kafka 中大量使用了<a href="../../../book-notes/OS.md">页缓存</a>，这是 Kafka 实现高吞吐的重要因素之一。虽然消息都是先被写入页缓存，然后由操作系统负责具体的刷盘任务的，但在Kafka中同样提供了同步刷盘及间断性强制刷盘（fsync）的功能，这些功能可以通过log.flush.interval.messages、log.flush.interval.ms 等参数来控制。同步刷盘可以提高消息的可靠性，防止由于机器掉电等异常造成处于页缓存而没有及时写入磁盘的消息丢失。不过笔者并不建议这么做，刷盘任务就应交由操作系统去调配，消息的可靠性应该由多副本机制来保障，而不是由同步刷盘这种严重影响性能的行为来保障。
- Linux系统会使用磁盘的一部分作为swap分区，这样可以进行进程的调度：把当前非活跃的进程调入 swap 分区，以此把内存空出来让给活跃的进程。对大量使用系统页缓存的 Kafka而言，应当尽量避免这种内存的交换，否则会对它各方面的性能产生很大的负面影响。我们可以通过修改vm.swappiness参数（Linux系统参数）来进行调节。vm.swappiness参数的上限为100，它表示积极地使用 swap 分区，并把内存上的数据及时地搬运到 swap 分区中；vm.swappiness 参数的下限为 0，表示在任何情况下都不要发生交换（vm.swappiness=0的含义在不同版本的 Linux 内核中不太相同，这里采用的是变更后的最新解释），这样一来，当内存耗尽时会根据一定的规则突然中止某些进程。笔者建议将这个参数的值设置为 1，这样保留了swap的机制而又最大限度地限制了它对Kafka性能的影响。

## 零拷贝
- Kafka还使用零拷贝（Zero-Copy）技术来进一步提升性能。所谓的零拷贝是指将数据直接从磁盘文件复制到网卡设备中，而不需要经由应用程序之手。零拷贝大大提高了应用程序的性能，减少了内核和用户模式之间的上下文切换。对 Linux操作系统而言，零拷贝技术依赖于底层的 sendfile（）方法实现。对应于 Java 语言，FileChannal.transferTo（）方法的底层实现就是sendfile（）方法。

## 传统IO描述
你需要将静态内容（类似图片、文件）展示给用户  
在这个过程中，文件A经历了4次复制的过程：
- （1）调用read（）时，文件A中的内容被复制到了内核模式下的Read Buffer中。
- （2）CPU控制将内核模式数据复制到用户模式下。
- （3）调用write（）时，将用户模式下的内容复制到内核模式下的Socket Buffer中。
- （4）将内核模式下的Socket Buffer的数据复制到网卡设备中传送。  

<img src="../../../imgs/非零拷贝IO.png" height=400px width=600px>

从上面的过程可以看出，数据平白无故地从内核模式到用户模式“走了一圈”，浪费了 2次复制过程：第一次是从内核模式复制到用户模式；第二次是从用户模式再复制回内核模式，即上面4次过程中的第2步和第3步。而且在上面的过程中，内核和用户模式的上下文的切换也是4次。

## 零拷贝
- 零拷贝技术通过DMA（Direct Memory Access）技术将文件内容复制到内核模式下的Read Buffer 中。不过没有数据被复制到Socket Buffer，相反只有包含数据的位置和长度的信息的文件描述符被加到Socket Buffer中。DMA引擎直接将数据从内核模式中传递到网卡设备（协议引擎）。这里数据只经历了2次复制就从磁盘中传送出去了，并且上下文切换也变成了2次。零拷贝是针对内核模式而言的，数据在内核模式下实现了零拷贝。 

<img src="../../../imgs/零拷贝IO.png" height=400px width=600px>

## tips
- Kafka官方并不建议通过Broker端的log.flush.interval.messages和log.flush.interval.ms来强制写盘，认为数据的可靠性应该通过Replica来保证，而强制Flush数据到磁盘会对整体性能产生影响。
- 可以通过调整/proc/sys/vm/dirty_background_ratio和/proc/sys/vm/dirty_ratio来调优性能。
- 脏页率超过第一个指标会启动pdflush开始Flush Dirty PageCache。
- 脏页率超过第二个指标会阻塞所有的写操作来进行Flush。
- 根据不同的业务需求可以适当的降低dirty_background_ratio和提高dirty_ratio。

# 消息传输保障
一般而言，消息中间件的消息传输保障有3个层级，分别如下。
- （1）at most once：至多一次。消息可能会丢失，但绝对不会重复传输。
- （2）at least once：最少一次。消息绝不会丢失，但可能会重复传输。
- （3）exactly once：恰好一次。每条消息肯定会被传输一次且仅传输一次。

## kafka消息传输机制
### producer
Kafka 的消息传输保障机制非常直观。当生产者向 Kafka 发送消息时，一旦消息被成功提交到日志文件，由于多副本机制的存在，这条消息就不会丢失。如果生产者发送消息到 Kafka之后，遇到了网络问题而造成通信中断，那么生产者就无法判断该消息是否已经提交。虽然Kafka无法确定网络故障期间发生了什么，但生产者可以进行多次重试来确保消息已经写入 Kafka，这个重试的过程中有可能会造成消息的重复写入，所以这里Kafka 提供的消息传输保障为 at least once。

### consumer
```对消费者而言```，消费者处理消息和提交消费位移的顺序在很大程度上决定了```消费者提供哪一种消息传输保障```。如果消费者在拉取完消息之后，应用逻辑先处理消息后提交消费位移，那么在消息处理之后且在位移提交之前消费者宕机了，待它重新上线之后，会从上一次位移提交的位置拉取，这样就出现了重复消费，因为有部分消息已经处理过了只是还没来得及提交消费位移，此时就对应at least once。如果消费者在拉完消息之后，应用逻辑先提交消费位移后进行消息处理，那么在位移提交之后且在消息处理完成之前消费者宕机了，待它重新上线之后，会从已经提交的位移处开始重新消费，但之前尚有部分消息未进行消费，如此就会发生消息丢失，此时就对应at most once。

## 狭义EOS语义实现
Kafka从0.11.0.0版本开始引入了幂等和事务这两个特性，以此来实现EOS（exactly once semantics，精确一次处理语义）。
所谓的幂等，简单地说就是对接口的多次调用所产生的结果和调用一次是一致的。生产者在进行重试的时候有可能会重复写入消息，而使用Kafka的幂等性功能之后就可以避免这种情况。

### 配置：
开启幂等性功能的方式很简单，只需要显式地将生产者客户端参数enable.idempotence设置为true即可,其他参数默认就行。此时acks 参数默认为"-1"，retries 参数默认大于0

### 原理
每个新的生产者实例在初始化的时候都会被分配一个PID，这个PID对用户而言是完全透明的。对于每个PID，消息发送到的每一个分区都有对应的序列号，这些序列号从0开始单调递增。生产者每发送一条消息就会将＜PID，分区＞对应的序列号的值加1。broker端会在内存中为每一对＜PID，分区＞维护一个序列号。对于收到的每一条消息，只有当它的序列号的值（SN_new）比broker端中维护的对应的序列号的值（SN_old）大1（即SN_new=SN_old+1）时，broker才会接收它。如果SN_new＜SN_old+1，那么说明消息被重复写入，broker可以直接将其丢弃。如果SN_new＞SN_old+1，那么说明中间有数据尚未写入，出现了乱序，暗示可能有消息丢失，对应的生产者会抛出OutOfOrderSequenceException。

引入序列号来实现幂等也只是针对每一对＜PID，分区＞而言的，也就是说，Kafka的幂等只能保证单个生产者会话（session）中单分区的幂等，Kafka 并不会保证消息内容的幂等(重复消息发送到不同的分区)。

## 事务
幂等性并不能跨多个分区运作，而事务可以弥补这个缺陷。事务可以保证对多个分区写入操作的原子性。操作的原子性是指多个操作要么全部成功，要么全部失败，不存在部分成功、部分失败的可能。
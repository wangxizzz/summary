1.**基本概念与原理介绍：**
- topic由一系列partition组成，为了提高容灾。topic具体路由到哪个partition，有routin算法决定。
- kafka是基于pull的方式，consumer端去拉broker端的消息。
- 通常，一个message将会被生产为一个特殊的topic.
- consumer端维护读消息的偏移量，不由broker维护。
- 一个简单的kafka服务器是一个broker.
- 

kafka启动
- 先启动自带zk：./zookeeper-server-start.sh -daemon ../config/zookeeper.properties
- 再启动kafka: ./kafka-server-start.sh -daemon ../config/server.properties(启动端口为9092,此时broker监听的端口是9092)
- 可以再启动zk-shell：  ./zookeeper-shell.sh localhost:2181
- 在kafka命令行创建topic: ./kafka-topics.sh --create --topic java_topic --zookeeper localhost:2181 --partitions 1 --replication-factor 1
- 命令行创建producer客户端:./kafka-console-producer.sh --topic test --broker-list 192.168.1.102:9092
- 命令行创建consumer客户端：./kafka-console-consumer.sh --topic test --bootstrap-serve 192.168.1.102:9092
- 命令行创建consumer客户端： ./kafka-console-consumer.sh --topic test --bootstrap-serve 192.168.1.102:9092 --from-beginning 加上--from-beginning可以从头开始pull生产者端的消息。
- ```注意：```**把配置文件中的所有localhost或者默认主机名都换成虚拟机的ip地址，这样可以减少很多windows与虚拟机通信的问题。**

HDD传统机械硬盘。SSD固态硬盘。
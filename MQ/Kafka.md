1.**基本概念与原理介绍：**
- topic由一系列partition组成，为了提高容灾。topic具体路由到哪个partition，有routin算法决定。
- kafka是基于pull的方式，consumer端去拉broker端的消息。
- 通常，一个message将会被生产为一个特殊的topic.
- consumer端维护读消息的偏移量，不由broker维护。
- 一个简单的kafka服务器是一个broker.
- 

kafka启动
- 先启动自带zk：./zookeeper-server-start.sh -daemon ../config/zookeeper.properties
- 再启动kafka: ./kafka-server-start.sh -daemon ../config/server.properties(启动端口为9092)
- 可以再启动zk-shell：  ./zookeeper-shell.sh localhost:2181


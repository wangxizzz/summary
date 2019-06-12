1.kafka启动
- 先启动自带zk：./zookeeper-server-start.sh -daemon ../config/zookeeper.properties
- 再启动kafka: ./kafka-server-start.sh -daemon ../config/server.properties(启动端口为9092)
- 可以再启动zk-shell：  ./zookeeper-shell.sh localhost:2181

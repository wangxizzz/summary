# 启动：
```bash
#启动nameServer
./mqnamesrv &

#启动broker
./mqbroker -n localhost:9876 &

# 关闭namesrv
sh bin/mqshutdown namesrv

# 关闭broker
sh bin/mqshutdown broker
```

# 关于Mac开机太久的问题：
1、启动Springboot或者普通maven项目会长时间卡住，即时配置了hosts  
2、RocketMQ正常启动，但是用client连接时，会死活连接不上。

> 针对以上问题，直接重启动脑即可。
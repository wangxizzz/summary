1.**启动zk的相关命令：**
- 启动：./zkServer.sh start
- 查看启动状态：./zkServer.sh status
- 客户端命令行连接：./zkCli.sh localhost:2181
    - 如果连不上，那就去掉后面的ip+端口，直接./zkCli.sh 回车即可。

2、**zk的zoo.cfg配置介绍**
```shell
# zk的服务器心跳时间，单位是ms
tickTime=2000
# 投票选举新leader的初始化时间
initLimit=10
# leader与follower心跳检测最大容忍时间响应超过syncLimit*tickTime，leader认为follower死掉，从服务器列表中删除follower.
syncLimit=5
# 数据目录
dataDir=/tmp/zookeeper
# the port at which the clients will connect
clientPort=2181
```

3.**zk安装的注意事项：**
- 下载完压缩包，解压，修改conf/zooxxx.cfg文件为zoo.cfg
- 然后打开zoo.cfg文件，配置zk的数据目录(数据节点存放的目录)

4、**zk的Java客户端操作注意事项：**
- zk的Java客户端API不支持递归创建节点。
- zk的Java客户端API删除节点时，只能删除叶子结点，不允许删除该节点至少存在一个子节点的情况。

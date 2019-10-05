1.**启动zk的相关命令：**
- 启动：./zkServer.sh start
- 查看启动状态：./zkServer.sh status
- 客户端命令行连接：./zkCli.sh -server IP:port
    - 如果连不上，那就去掉后面的ip+端口，直接./zkCli.sh 回车即可, 百试百灵(🤷‍♀️)。

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

5.**会话(Session):**
- 使用客户端来创建一个和zk服务端连接的句柄，这就是一个会话（session）。Session一旦建立，状态就是连接中（CONNECTING）状态，然后客户端会尝试去连接zk服务端，连接成功之后状态变成已连接（CONNECTED）。一般正常情况下只会有这两个状态。不过，还是会发生一些无法恢复的错误/故障，比如：session过期，认证失败，或者客户端关闭连接，这种情况下，session状态会变成关闭（CLOSED）状态。
- 使用zk客户端创建session的一个参数是session超时时间（毫秒）。但是请注意，这个session超时时间并不是客户端可以随意设置的。zk客户端会把这个session超时时间发给服务端，服务端会返回一个他可以接受的值给客户端。标准其实就是tickTime*2 <= session timeout <= tickTime*20。
如果zk客户端和zk服务端集群断开连接之后，在session超时时间之内，重新连接上了，那么session状态重新变为connected，如果在session超时时间之内没有连接上，那么session状态会变成expired。当session断开连接的时候，最好不用自己去建立一个新的session，因为zk客户端已经帮我做了这个工作，他会自动重连的。只有一种情况需要我们手动重新创建新的session，那就是明确知道session状态为过期状态（expiration）。

6.**数据节点ZNode：**
- 节点类型：
    - 永久节点：该节点的生命周期不依赖于会话，并且只有在客户端显式执行删除操作时，它们才能被删除。
    - 临时节点：该节点的生命周期依赖于创建它的会话，一旦会话结束，临时节点将被自动删除，也可以手动删除。临时节点没有子节点
    - 序列节点：Znode节点还有一个序列化的特性， 如果创建时指明为序列化节点，则该节点被创建时，节点名后面默认跟10位序列号，序列号对于此父节点来说是唯一的。
- **zk规定，所有非叶子节点必须是持久化节点。**
- 数据结构,对于每个Znode,zookeeper都会维护一个Stat数据结构：
    - 每个ZNode都可以保存数据，同时还挂载着子节点。
    - Stat对象重要状态说明：
        - version : 数据节点版本号
        - cversion : 子节点版本号
        - dataLength: 数据内容的长度
        - numChildren : 当前节点子节点的个数
        - mtime : 表示该节点最后一次更新的时间
        - czxid,mzxid ： 分别表示节点创建的事务id与节点最后一次被更新时的事务id
- Watcher----数据变更通知
    - 在zk中，接口类Watcher表示一个标准的事件处理器，其定义了事件通知机制，包含KeeperState与EventType两个枚举类，分别表示通知状态与事件类型，同时定义了事件回调方法process(WatcherEvent event),当zk向客户端发送一个Watcher事件通知时，客户端会回调此方法，从而对事件进行处理。

7.**zookeeper的应用场景介绍：**
- 数据发布与订阅，即所谓的配置中心。
    - 发布者把把数据发布到zk的节点上，供订阅者来订阅。发布订阅有推拉模式，zk两种模式混合使用：客户端向服务端注册自己感兴趣的节点，一旦该节点数据发生变化，服务会向相应的客户端发送watcher事件通知，客户端收到这个通知后，会向服务端拉取最新的配置数据。


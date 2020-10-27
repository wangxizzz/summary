# 心跳作用
dubbo默认使用Netty作为通讯工具，Netty在消费端和服务端之间建立长连接。当建立长连接后，需要使用心跳机制判断双方是否在线。

检测对方是否在线是长连接非常重要的一种机制。如果不检测，那么自身是无法感知对方掉线的，对方一旦掉线了，长连接也就断开了，双方通讯无法正常完成。有些检测机制是通过发送心跳报文完成，有些检测机制是通过发送业务请求时，顺带检测是否在线。

dubbo使用心跳机制检测对方是否在线.

# 实现原理
dubbo的心跳发送是由Netty的IdleStateHandler对象处理的。该对象是一个handler，配置在netty的责任链里面，当发送请求或者收到响应时，都会经过该对象处理。在双方通讯开始后该对象会创建一些空闲检测定时器，用于检测读事件（收到请求会触发读事件）和写事件（连接、发送请求会触发写事件）。当在指定的空闲时间内没有收到读事件或写事件，便会触发超时事件，然后IdleStateHandler将超时事件交给责任链里面的下一个handler NettyClientHandler或者NettyServertHandler处理，NettyClientHandler是客户端使用的，它收到事件后向对方发送一个心跳事件的请求，NettyServertHandler是服务端使用的，它收到事件后将关闭超时的连接。
dubbo还提供了HeaderExchangeClient。HeaderExchangeClient在客户端使用，当发现连接在指定时间内没有收到响应报文，而且连接已经不可用，那么会启动重连机制，与服务端重新建立连接。

# dubbo框架的心跳机制
注意：dubbo框架本身提供了心跳机制，当底层的通信框架不支持心跳检测时，即会使用dubbo自身的心跳检测(源码分析会体现这一判断)。当底层通信框架是netty时，则采用netty本身的检测机制。
## dubbo consumer端
```java
// 在开启nettyClient，建立连接时，会创建HeaderExchangeClient
public class HeaderExchangeClient implements ExchangeClient {

    private final Client client;
    private final ExchangeChannel channel;

    // 执行心跳与重连任务超时的时间轮机制
    private static final HashedWheelTimer IDLE_CHECK_TIMER = new HashedWheelTimer(
            new NamedThreadFactory("dubbo-client-idleCheck", true), 1, TimeUnit.SECONDS, TICKS_PER_WHEEL);
    private HeartbeatTimerTask heartBeatTimerTask;
    private ReconnectTimerTask reconnectTimerTask;

    // 调用构造函数
    public HeaderExchangeClient(Client client, boolean startTimer) {
        Assert.notNull(client, "Client can't be null");
        this.client = client;
        this.channel = new HeaderExchangeChannel(client);
        // startTimer为true
        if (startTimer) {
            URL url = client.getUrl();
            // 开启重连和 心跳任务
            startReconnectTask(url);
            startHeartBeatTask(url);
        }
    }
    .....
}
// 开启重连的超时任务
private void startReconnectTask(URL url) {
    // 重连超时任务默认是开启的
    if (shouldReconnect(url)) {
        AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);
        int idleTimeout = getIdleTimeout(url);
        long heartbeatTimeoutTick = calculateLeastDuration(idleTimeout);
        this.reconnectTimerTask = new ReconnectTimerTask(cp, heartbeatTimeoutTick, idleTimeout);
        IDLE_CHECK_TIMER.newTimeout(reconnectTimerTask, heartbeatTimeoutTick, TimeUnit.MILLISECONDS);
    }
}
private boolean shouldReconnect(URL url) {
    return url.getParameter(Constants.RECONNECT_KEY, true);
}

// 开启发送心跳任务
private void startHeartBeatTask(URL url) {
    // 如果底层通信的 client支持空闲检测，那么跳过dubbo自身的心跳检测机制.因此dubbo的心跳机制采用的Netty带的
    if (!client.canHandleIdle()) {
        AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);
        int heartbeat = getHeartbeat(url);
        long heartbeatTick = calculateLeastDuration(heartbeat);
        this.heartBeatTimerTask = new HeartbeatTimerTask(cp, heartbeatTick, heartbeat);
        IDLE_CHECK_TIMER.newTimeout(heartBeatTimerTask, heartbeatTick, TimeUnit.MILLISECONDS);
    }
}
// 看下 client.canHandleIdle()
// org.apache.dubbo.remoting.transport.netty4.NettyClient#canHandleIdle
@Override
public boolean canHandleIdle() {
    return true;
}

// 时间轮执行超时任务
// org.apache.dubbo.remoting.exchange.support.header.AbstractTimerTask#run
public void run(Timeout timeout) throws Exception {
    Collection<Channel> c = channelProvider.getChannels();
    for (Channel channel : c) {
        if (channel.isClosed()) {
            continue;
        }
        doTask(channel);
    }
    // 超时任务重放入时间轮，设置下一次的超时任务
    reput(timeout, tick);
}
```
## dubbo超时任务的具体做法
```java
// 心跳任务
// org.apache.dubbo.remoting.exchange.support.header.HeartbeatTimerTask#doTask
protected void doTask(Channel channel) {
    try {
        Long lastRead = lastRead(channel);
        Long lastWrite = lastWrite(channel);
        if ((lastRead != null && now() - lastRead > heartbeat)
                || (lastWrite != null && now() - lastWrite > heartbeat)) {
            Request req = new Request();
            req.setVersion(Version.getProtocolVersion());
            req.setTwoWay(true);
            req.setEvent(HEARTBEAT_EVENT);
            channel.send(req);
            logger.info("Send heartbeat to remote channel " + channel.getRemoteAddress()
                    + ", cause: The channel has no data-transmission exceeds a heartbeat period: "
                    + heartbeat + "ms");
        }
    } catch (Throwable t) {
        logger.warn("Exception when heartbeat to remote channel " + channel.getRemoteAddress(), t);
    }
}

// 重连任务
// org.apache.dubbo.remoting.exchange.support.header.ReconnectTimerTask#doTask
protected void doTask(Channel channel) {
    try {
        Long lastRead = lastRead(channel);
        Long now = now();

        // Rely on reconnect timer to reconnect when AbstractClient.doConnect fails to init the connection
        if (!channel.isConnected()) {
            try {
                // 注意：如果使用dubbo的provider优雅关闭，是不会走到这来的。provider优雅关闭会清空zk的provider节点，并且会refresh consumer端的invoker列表，因此consumer端重连是没有关闭的provider IP的。
                // 如果直接使用kill -9 pid 杀掉provider进程，那么会进入此进行重连，此时会走进exception代码块，目前看此过程走了3次，就自动取消了zk对应的provider订阅，不再重试。
                logger.info("Initial connection to " + channel);
                ((Client) channel).reconnect();
            } catch (Exception e) {
                logger.error("Fail to connect to " + channel, e);
            }
        // check pong at client
        } else if (lastRead != null && now - lastRead > idleTimeout) {
            logger.warn("Reconnect to channel " + channel + ", because heartbeat read idle time out: "
                    + idleTimeout + "ms");
            try {
                ((Client) channel).reconnect();
            } catch (Exception e) {
                logger.error(channel + "reconnect failed during idle time.", e);
            }
        }
    } catch (Throwable t) {
        logger.warn("Exception when reconnect to remote channel " + channel.getRemoteAddress(), t);
    }
}
```


## dubbo provider端
```java
// 在开启 Netty Server 绑定端口时，会启动provider端空闲检测任务
// org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeServer
public HeaderExchangeServer(RemotingServer server) {
    Assert.notNull(server, "server == null");
    this.server = server;
    startIdleCheckTask(getUrl());
}

// startIdleCheckTask
private void startIdleCheckTask(URL url) {
    // 利用Netty自身的，跳过dubbo本身自带的空闲检测
    if (!server.canHandleIdle()) {
        AbstractTimerTask.ChannelProvider cp = () -> unmodifiableCollection(HeaderExchangeServer.this.getChannels());
        int idleTimeout = getIdleTimeout(url);
        long idleTimeoutTick = calculateLeastDuration(idleTimeout);
        CloseTimerTask closeTimerTask = new CloseTimerTask(cp, idleTimeoutTick, idleTimeout);
        this.closeTimerTask = closeTimerTask;

        // init task and start timer.
        IDLE_CHECK_TIMER.newTimeout(closeTimerTask, idleTimeoutTick, TimeUnit.MILLISECONDS);
    }
}

// org.apache.dubbo.remoting.transport.netty4.NettyServer#canHandleIdle
@Override
public boolean canHandleIdle() {
    // netty支持服务端空闲检测
    return true;
}

// provider空闲检测具体做法
@Override
protected void doTask(Channel channel) {
    try {
        Long lastRead = lastRead(channel);
        Long lastWrite = lastWrite(channel);
        Long now = now();
        // check ping & pong at server
        if ((lastRead != null && now - lastRead > idleTimeout)
                || (lastWrite != null && now - lastWrite > idleTimeout)) {
            logger.warn("Close channel " + channel + ", because idleCheck timeout: "
                    + idleTimeout + "ms");
            // 直接断开连接
            channel.close();
        }
    } catch (Throwable t) {
        logger.warn("Exception when close remote channel " + channel.getRemoteAddress(), t);
    }
}
```

# netty的心跳检测
```java
// 首先，必须添加IdleStateHandler
// org.apache.dubbo.remoting.transport.netty4.NettyServer 
ch.pipeline()
        .addLast("decoder", adapter.getDecoder())
        .addLast("encoder", adapter.getEncoder())
        .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
        .addLast("handler", nettyServerHandler);

// org.apache.dubbo.remoting.transport.netty4.NettyClient
ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
            .addLast("decoder", adapter.getDecoder())
            .addLast("encoder", adapter.getEncoder())
            .addLast("client-idle-handler", new IdleStateHandler(heartbeatInterval, 0, 0, MILLISECONDS))
            .addLast("handler", nettyClientHandler);
```

https://blog.csdn.net/weixin_38308374/article/details/106736714
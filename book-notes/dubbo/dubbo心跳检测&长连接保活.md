# 心跳作用
dubbo默认使用Netty作为通讯工具，Netty在消费端和服务端之间建立长连接。当建立长连接后，需要使用心跳机制判断双方是否在线。
```通过心跳机制来维护长连接，保活```

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
    // 超时重连任务默认是开启的，当采用Netty的心跳机制时，仍然是开启的
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

# Netty的心跳检测
```dubbo默认使用netty的心跳机制```
```java
// 首先，必须添加IdleStateHandler
// org.apache.dubbo.remoting.transport.netty4.NettyServer 
ch.pipeline()
        .addLast("decoder", adapter.getDecoder())
        .addLast("encoder", adapter.getEncoder())
        .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
        .addLast("handler", nettyServerHandler);

// cleint端的超时，发送心跳的时间设置
public static int getHeartbeat(URL url) {
    return url.getParameter(Constants.HEARTBEAT_KEY, Constants.DEFAULT_HEARTBEAT);
}

// org.apache.dubbo.remoting.transport.netty4.NettyClient
ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
            .addLast("decoder", adapter.getDecoder())
            .addLast("encoder", adapter.getEncoder())
            .addLast("client-idle-handler", new IdleStateHandler(heartbeatInterval, 0, 0, MILLISECONDS))
            .addLast("handler", nettyClientHandler);

// server端的超时，发送心跳的时间设置
public static int getIdleTimeout(URL url) {
    // 可以看出server的时间默认 是 client的 3倍
    int heartBeat = getHeartbeat(url);
    // idleTimeout should be at least more than twice heartBeat because possible retries of client.
    int idleTimeout = url.getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartBeat * 3);
    if (idleTimeout < heartBeat * 2) {
        throw new IllegalStateException("idleTimeout < heartbeatInterval * 2");
    }
    return idleTimeout;
}
```
## IdleStateHandler 分析
```java
 public IdleStateHandler(
        long readerIdleTime, long writerIdleTime, long allIdleTime,
        TimeUnit unit) {
    this(false, readerIdleTime, writerIdleTime, allIdleTime, unit);
}
```

我们先来看一下IdleStateHandler构造方法各个入参的含义：
- readerIdleTime：读空闲超时检测定时任务会在每readerIdleTime时间内启动一次，检测在readerIdleTime内是否发生过读事件，如果没有发生过，则触发读超时事件READER_IDLE_STATE_EVENT，并将超时事件交给NettyClientHandler处理。如果为0，则不创建定时任务。
- writerIdleTime：与readerIdleTime作用类似，只不过该参数定义的是写事件。
- allIdleTime：同时检测读事件和写事件，如果在allIdleTime时间内即没有发生过读事件，也没有发生过写事件，则触发超时事件ALL_IDLE_STATE_EVENT。
- unit：表示前面三个参数的单位，就上面代码来说，表示的是毫秒。

服务端创建的IdleStateHandler设置了allIdleTime，所以服务端的定时任务需要检测读事件和写事件。定时任务的启动时间间隔是参数“heartbeat”设置值的3倍，heartbeat默认是1分钟，也可以通过参数“heartbeat.timeout”设置定时任务的启动时间间隔。单位都是毫秒。

NettyClient创建IdleStateHandler只设置了readerIdleTime入参，表示客户端启动定时任务只检测读事件。定时任务的时间间隔由参数“heartbeat”指定，默认是1分钟。

当Channel被激活的时候，也就是连接被建立起来之后，会调用IdleStateHandler的initialize方法：
```java
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    initialize(ctx);
    super.channelActive(ctx);
}

// 跟进initialize方法
private void initialize(ChannelHandlerContext ctx) {
    switch (state) {
    case 1:
    case 2:
        return;
    }

    state = 1;
    initOutputChanged(ctx);
    // 连接初次建立时，初始化成员变量 lastReadTime = lastWriteTime为当前系统的纳秒时间
    lastReadTime = lastWriteTime = ticksInNanos();
    if (readerIdleTimeNanos > 0) {
        // 具体由Nio线程进行定时调度 超时任务的处理
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                writerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                allIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
}
// ticksInNanos 获取纳秒
long ticksInNanos() {
    return System.nanoTime();
}

// 我们分析lastReadTime，lastWriteTime 在哪里改变的

// 更新lastReadTime
@Override
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
    if ((readerIdleTimeNanos > 0 || allIdleTimeNanos > 0) && reading) {
        // 读操作完毕，更新lastReadTime，并且设置reading标志位false，表示不在读取中
        lastReadTime = ticksInNanos();
        reading = false;
    }
    ctx.fireChannelReadComplete();
}
// 更新lastWriteTime时间
private final ChannelFutureListener writeListener = new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        lastWriteTime = ticksInNanos();
        firstWriterIdleEvent = firstAllIdleEvent = true;
    }
};

// 超时任务的调度，采用线程池的调度
ScheduledFuture<?> schedule(ChannelHandlerContext ctx, Runnable task, long delay, TimeUnit unit) {
    return ctx.executor().schedule(task, delay, unit);
}

// 看下 读事件超时的run方法
// io.netty.handler.timeout.IdleStateHandler.ReaderIdleTimeoutTask#run
protected void run(ChannelHandlerContext ctx) {
    long nextDelay = readerIdleTimeNanos;
    if (!reading) {
        // reading为false表示 读取完毕置为false
        // 如果 很长时间channel没有发生读事件，那么ticksInNanos() - lastReadTime 就会很大即会超过设置的 读超时时间readerIdleTimeNanos，因此就会触发心跳事件的发送
        nextDelay -= ticksInNanos() - lastReadTime;
    }

    if (nextDelay <= 0) {
        // Reader is idle - set a new timeout and notify the callback.
        readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);

        boolean first = firstReaderIdleEvent;
        firstReaderIdleEvent = false;

        try {
            // 触发读读超时的心跳事件触发
            IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
            // 把事件发送给 下一个Handler --> NettyClientHandler/NettyServerHandler,做不同的操作
            channelIdle(ctx, event);
        } catch (Throwable t) {
            ctx.fireExceptionCaught(t);
        }
    } else {
        // Read occurred before the timeout - set a new timeout with shorter delay.
        readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
}

// io.netty.handler.timeout.IdleStateHandler#channelIdle
protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
    ctx.fireUserEventTriggered(evt);
}
```

```注意：保持长连接，是由client端发起的心跳 Request请求。因为client的IdleHandler设置的时间比server端小```

NettyClientHandler/NettyServerHandler 对心跳事件做的处理：
```java
// client 端
// org.apache.dubbo.remoting.transport.netty4.NettyClientHandler#userEventTriggered
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
    // send heartbeat when read idle.
    // 收到client端自己的 IdleHandler的心跳事件，然后发送Request请求 到server端，以此来更新server端的channelReadTime，从而不会触发server端的IdleHandler的心跳超时任务，从而不会触发server端关闭channel的操作。
    if (evt instanceof IdleStateEvent) {
        try {
            NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
            if (logger.isDebugEnabled()) {
                logger.debug("IdleStateEvent triggered, send heartbeat to channel " + channel);
            }
            // 封装请求发送心跳事件, 服务端接收到心跳，会更新读时间以此来维护存活的长连接
            Request req = new Request();
            req.setVersion(Version.getProtocolVersion());
            req.setTwoWay(true);
            req.setEvent(HEARTBEAT_EVENT);
            channel.send(req);
        } finally {
            // 判断如果服务端 channel关闭，那么client端会移除channel 
            NettyChannel.removeChannelIfDisconnected(ctx.channel());
        }
    } else {
        super.userEventTriggered(ctx, evt);
    }
}


// Server端
// org.apache.dubbo.remoting.transport.netty4.NettyServerHandler#userEventTriggered
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
    // server will close channel when server don't receive any heartbeat from client util timeout.
    // 当服务端收到server端自己的 IdleHandler发送的心跳事件，此时server端自己的 IdleHandler的channelReadTime一直没更新，此时server端就会认为client已经掉线了，就会关闭client端的channel。因为此时client端即没正常的请求，也没有心跳的Request请求，从而不会更新server的channelReadTime.
    if (evt instanceof IdleStateEvent) {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        try {
            logger.info("IdleStateEvent triggered, close channel " + channel);
            channel.close();
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.channel());
        }
    }
    super.userEventTriggered(ctx, evt);
}
```

# 四、总结
当服务端发生超时事件后，服务端会将对应的连接关闭。  

当客户端发生超时事件后，客户端通过超时重连以及发送心跳尝试维持连接。  

服务端和客户端对超时后作出的不同操作也反映了双方不同的策略。因为连接占用系统资源，服务端要尽可能的将资源留给其他请求，对于服务端来说，如果某个连接长时间没有数据传输，说明与该客户端的连接已经断开，或者客户端访问已经结束最近不需要再次访问，无论哪种情况，对于服务端来说最好的处理都是断开与客户端的连接。  

```而客户端则不同，客户端想尽全力保证连接的可用，因为客户端访问服务时最希望的是尽快得到响应，因此客户端最好是时时刻刻保持连接的可用，这样访问服务时可以省去建立连接的时间消耗。  ```

另外一点也要主要，服务端和客户端启动定时任务的时间是不同的，默认服务端是3分钟，客户端是1分钟，dubbo要求服务端定时任务的启动时间间隔最小是客户端的2倍。  

参考：https://dubbo.apache.org/zh-cn/blog/dubbo-heartbeat-design/  有的可能过时了
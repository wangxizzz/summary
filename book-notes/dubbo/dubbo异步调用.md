# 原理
Dubbo的服务消费端基于CompletableFuture实现了纯异步调用，其实还不单单CompletableFuture的功劳，归根到底是Netty的NIO非阻塞功能提供的底层实现。

# 消费方 发起请求
```java
// org.apache.dubbo.rpc.protocol.AbstractInvoker#invoke
public Result invoke(Invocation inv) throws RpcException {
    ....
    RpcInvocation invocation = (RpcInvocation) inv;
    invocation.setInvoker(this);
    if (CollectionUtils.isNotEmptyMap(attachment)) {
        invocation.addObjectAttachmentsIfAbsent(attachment);
    }

    Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
    ....

    invocation.setInvokeMode(RpcUtils.getInvokeMode(url, invocation));
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

    AsyncRpcResult asyncResult;
    try {
        // 调用执行方法
        asyncResult = (AsyncRpcResult) doInvoke(invocation);
    } catch (InvocationTargetException e) { // biz exception
        ....
    } catch (Throwable e) {
        asyncResult = AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
    }
    RpcContext.getContext().setFuture(new FutureAdapter(asyncResult.getResponseFuture()));
    return asyncResult;
}
```
跟进doInvoke方法：

```java
// org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke
@Override
protected Result doInvoke(final Invocation invocation) throws Throwable {
    // RpcInvocation包含了调用需要的参数(方法名、参数等)
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(PATH_KEY, getUrl().getPath());
    inv.setAttachment(VERSION_KEY, version);
    // clients是ExchangeClient数组，ExchangeClient本质是 NettyClient，而且clients的length大部分为1(即使一台机器起多个consumer端，length也是1)
    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        int timeout = calculateTimeout(invocation, methodName);
        invocation.put(TIMEOUT_KEY, timeout);
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            return AsyncRpcResult.newDefaultAsyncResult(invocation);
        } else {
            ExecutorService executor = getCallbackExecutor(getUrl(), inv);
            // 发送request请求server端，返回 CompletableFuture, 释放业务线程
            CompletableFuture<AppResponse> appResponseFuture =
                    currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
            // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
            FutureContext.getContext().setCompatibleFuture(appResponseFuture);
            AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
            result.setExecutor(executor);
            return result;
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```
我们继续跟进request方法：

```java
// org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel#request(java.lang.Object, int, java.util.concurrent.ExecutorService)
@Override
public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
    // request就是RpcInvocation
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
    }
    // create request. 把RpcInvocation封装成新的Request对象
    Request req = new Request();
    req.setVersion(Version.getProtocolVersion());
    req.setTwoWay(true);
    req.setData(request);  // 设置Data
    // 返回Future，并且把生成的Future存入 全局的Map中
    DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout, executor);
    try {
        // 利用netty发送
        channel.send(req);
    } catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}

// 跟进DefaultFuture.newFuture
// org.apache.dubbo.remoting.exchange.support.DefaultFuture#newFuture
public static DefaultFuture newFuture(Channel channel, Request request, int timeout, ExecutorService executor) {
    final DefaultFuture future = new DefaultFuture(channel, request, timeout);
    future.setExecutor(executor);
    // ThreadlessExecutor needs to hold the waiting future in case of circuit return.
    if (executor instanceof ThreadlessExecutor) {
        ((ThreadlessExecutor) executor).setWaitingFuture(future);
    }
    // dubbo框架中 方法调用的超时处理。下面分析
    timeoutCheck(future);
    return future;
}

// DefaultFuture全局Map
private static final Map<Long, DefaultFuture> FUTURES = new ConcurrentHashMap<>();

// DefaultFuture构造函数
private DefaultFuture(Channel channel, Request request, int timeout) {
    this.channel = channel;
    this.request = request;
    this.id = request.getId();
    this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
    // put into waiting map.  放入等待的map中
    FUTURES.put(id, this);
    CHANNELS.put(id, channel);
}
```
跟进channel.send
```java
// org.apache.dubbo.remoting.transport.AbstractPeer#send
public void send(Object message) throws RemotingException {
    send(message, url.getParameter(Constants.SENT_KEY, false));
}

// org.apache.dubbo.remoting.transport.AbstractClient#send
@Override
public void send(Object message, boolean sent) throws RemotingException {
    if (needReconnect && !isConnected()) {
        connect();
    }
    Channel channel = getChannel();
    if (channel == null || !channel.isConnected()) {
        throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
    }
    channel.send(message, sent);
}
// org.apache.dubbo.remoting.transport.netty4.NettyChannel#send
public void send(Object message, boolean sent) throws RemotingException {
    // whether the channel is closed
    super.send(message, sent);

    boolean success = true;
    int timeout = 0;
    try {
        // 通过NettyChannel 发送数据到channel
        // writeAndFlush会释放掉业务线程，利用netty的nio thread处理
        ChannelFuture future = channel.writeAndFlush(message);
        if (sent) {
            // wait timeout ms
            timeout = getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
            success = future.await(timeout);
        }
        Throwable cause = future.cause();
        if (cause != null) {
            throw cause;
        }
    } catch (Throwable e) {
        removeChannelIfDisconnected(channel);
        throw new RemotingException(this, "Failed to send message " + PayloadDropper.getRequestWithoutData(message) + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
    }
    if (!success) {
        throw new RemotingException(this, "Failed to send message " + PayloadDropper.getRequestWithoutData(message) + " to " + getRemoteAddress()
                + "in timeout(" + timeout + "ms) limit");
    }
}
```
consumer端数据发送到channel，后续会经过一系列的 OutBoundHandler

# producer端 接收请求并执行调用，发送请求结果到consumer
解码后的入口：
```java
// 入口：org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#received
public void received(Channel channel, Object message) throws RemotingException {
        final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
        if (message instanceof Request) {
            // handle request.
            Request request = (Request) message;
            if (request.isEvent()) {
                handlerEvent(channel, request);
            } else {
                if (request.isTwoWay()) {
                    // 来自consumer端的request会经过这里
                    handleRequest(exchangeChannel, request);
                } else {
                    handler.received(exchangeChannel, request.getData());
                }
            }
        } else if (message instanceof Response) {
            handleResponse(channel, (Response) message);
        }
        .....
}
// 跟进handleRequest方法
void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
    ....

    // find handler by message class.
    Object msg = req.getData();
    try {
        // provider端异步执行
        CompletionStage<Object> future = handler.reply(channel, msg);
        future.whenComplete((appResult, t) -> {
            try {
                if (t == null) {
                    // 设置 response的状态为 OK
                    res.setStatus(Response.OK);
                    res.setResult(appResult);
                } else {
                    res.setStatus(Response.SERVICE_ERROR);
                    res.setErrorMessage(StringUtils.toString(t));
                }
                // 当provider端执行结束，发送到channel到consumer端
                channel.send(res);
            } catch (RemotingException e) {
                logger.warn("Send result to consumer failed, channel is " + channel + ", msg is " + e);
            }
        });
    }
}

// 跟进reply方法
// org.apache.dubbo.remoting.exchange.support.ExchangeHandlerAdapter#reply
public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {
    if (!(message instanceof Invocation)) {
        throw new RemotingException(channel, "Unsupported request: "
                + (message == null ? null : (message.getClass().getName() + ": " + message))
                + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
    }
    // 获取反序列化后的请求参数等
    Invocation inv = (Invocation) message;
    Invoker<?> invoker = getInvoker(channel, inv);
    // need to consider backward-compatibility if it's a callback
    if (Boolean.TRUE.toString().equals(inv.getObjectAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
        String methodsStr = invoker.getUrl().getParameters().get("methods");

        ......
    }
    
    RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
    // 真正执行provider调用
    // org.apache.dubbo.rpc.proxy.AbstractProxyInvoker#invoke
    Result result = invoker.invoke(inv);
    // thenApply为Future get结果时的回调函数, thenApply为异步执行，不阻塞当前线程
    return result.thenApply(Function.identity());
}

// AbstractProxyInvoker#invoke
public Result invoke(Invocation invocation) throws RpcException {
        try {
    // 调用 字节码生成的Wrapper类的方法
    Object value = doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments());
    // 包装为 CompletableFuture
    CompletableFuture<Object> future = wrapWithFuture(value);
    CompletableFuture<AppResponse> appResponseFuture = future.handle((obj, t) -> {
        AppResponse result = new AppResponse();
        if (t != null) {
            if (t instanceof CompletionException) {
                result.setException(t.getCause());
            } else {
                result.setException(t);
            }
        } else {
            result.setValue(obj);
        }
        return result;
    });
    return new AsyncRpcResult(appResponseFuture, invocation);
    .....
}
```

# consumer端接收provider端返回的结果数据
首先 NettyClient端需要经过一系列的InBoundHandler处理，然后到达NettyClientHandler，最终的到达
org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#received
```java
// 入口： org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#received
public void received(Channel channel, Object message) throws RemotingException {
    final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    // 处理consumer端过来的Request
    if (message instanceof Request) {
        // handle request.
        Request request = (Request) message;
        if (request.isEvent()) {
            handlerEvent(channel, request);
        } else {
            if (request.isTwoWay()) {
                handleRequest(exchangeChannel, request);
            } else {
                handler.received(exchangeChannel, request.getData());
            }
        }
    } 
    // 处理provider端 的响应结果
    else if (message instanceof Response) {
        // 此时会进入这里
        handleResponse(channel, (Response) message);
    } else if (message instanceof String) {
        if (isClientSide(channel)) {
            Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
            logger.error(e.getMessage(), e);
        } else {
            String echo = handler.telnet(channel, (String) message);
            if (echo != null && echo.length() > 0) {
                channel.send(echo);
            }
        }
    } else {
        handler.received(exchangeChannel, message);
    }
}

// 跟进handleResponse
static void handleResponse(Channel channel, Response response) throws RemotingException {
    if (response != null && !response.isHeartbeat()) {
        DefaultFuture.received(channel, response);
    }
}

// org.apache.dubbo.remoting.exchange.support.DefaultFuture#received(org.apache.dubbo.remoting.Channel, org.apache.dubbo.remoting.exchange.Response, boolean)
public static void received(Channel channel, Response response, boolean timeout) {
    try {
        // 在consumer请求provider时，会生成一个Future，然后立马返回。并且会把这个Future放入DefaultFuture的全局Map中。当 NettyClient接收到了 provider的返回结果时，会从Map中取出 request时 返回给上层的Future，并且移除。然后设置这个Future的结果，那么在consumer端的上层获取的就是有结果的Future进行后续操作。
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
            Timeout t = future.timeoutCheckTask;
            if (!timeout) {
                // decrease Time
                t.cancel();
            }
            // 设置Future的结果
            future.doReceived(response);
        } else {
            logger.warn("The timeout response finally returned at "
                    + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                    + ", response status is " + response.getStatus()
                    + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                    + " -> " + channel.getRemoteAddress()) + ", please check provider side for detailed result.");
        }
    } finally {
        CHANNELS.remove(response.getId());
    }
}

// 跟进doReceived方法
private void doReceived(Response res) {
    if (res == null) {
        throw new IllegalStateException("response cannot be null");
    }
    if (res.getStatus() == Response.OK) {
        // 设置consumer端请求时返回给上层的Future的结果，并把执行状态置为完成。后续就会经过一系列Invoker，比如AsyncToSyncInvoker
        this.complete(res.getResult());
    } else if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
        this.completeExceptionally(new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage()));
    } else {
        this.completeExceptionally(new RemotingException(channel, res.getErrorMessage()));
    }

    // the result is returning, but the caller thread may still waiting
    // to avoid endless waiting for whatever reason, notify caller thread to return.
    if (executor != null && executor instanceof ThreadlessExecutor) {
        ThreadlessExecutor threadlessExecutor = (ThreadlessExecutor) executor;
        if (threadlessExecutor.isWaiting()) {
            threadlessExecutor.notifyReturn(new IllegalStateException("The result has returned, but the biz thread is still waiting" +
                    " which is not an expected state, interrupt the thread manually by returning an exception."));
        }
    }
}
```
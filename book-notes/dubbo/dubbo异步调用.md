Dubbo的服务消费端基于CompletableFuture实现了纯异步调用，其实还不单单CompletableFuture的功劳，归根到底是Netty的NIO非阻塞功能提供的底层实现。

# 消费方
```java
// org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke
@Override
protected Result doInvoke(final Invocation invocation) throws Throwable {
    // RpcInvocation包含了调用需要的参数(方法名、参数等)
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    inv.setAttachment(PATH_KEY, getUrl().getPath());
    inv.setAttachment(VERSION_KEY, version);

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

# producer端
解码后的入口：
```java
// org.apache.dubbo.remoting.exchange.support.ExchangeHandlerAdapter#received
public void received(Channel channel, Object message) throws RemotingException {
    if (message instanceof Invocation) {
        reply((ExchangeChannel) channel, message);

    } else {
        super.received(channel, message);
    }
}


public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {

    if (!(message instanceof Invocation)) {
        throw new RemotingException(channel, "Unsupported request: "
                + (message == null ? null : (message.getClass().getName() + ": " + message))
                + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
    }

    Invocation inv = (Invocation) message;
    Invoker<?> invoker = getInvoker(channel, inv);
    ...
    RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
    // 真正执行
    Result result = invoker.invoke(inv);
    // thenApply为Future get结果时的回调函数
    return result.thenApply(Function.identity());

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
# 粗略分析over
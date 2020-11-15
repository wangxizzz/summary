# 超时检测的时机
超时检测是发生在 consumer端发送请求到provider端。注意：超时检测是发生在consumer端，与provider端没有任何关系。

# 超时异常源码分析
```java
// 上层调用方：
// org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel#request(java.lang.Object, int, java.util.concurrent.ExecutorService)

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
```
```timeoutCheck```即为超时检测机制

```java
// 跟进dubbo方法调用超时处理
// org.apache.dubbo.remoting.exchange.support.DefaultFuture#timeoutCheck
/**
* check time out of the future
*/
private static void timeoutCheck(DefaultFuture future) {
    TimeoutCheckTask task = new TimeoutCheckTask(future.getId());
    // 利用时间轮机制来控制 调用是否超时。 future.getTimeout()代表方法调用的超时时间
    future.timeoutCheckTask = TIME_OUT_TIMER.newTimeout(task, future.getTimeout(), TimeUnit.MILLISECONDS);
}
// 处理超时的时间轮
public static final Timer TIME_OUT_TIMER = new HashedWheelTimer(
            new NamedThreadFactory("dubbo-future-timeout", true),
            30,
            TimeUnit.MILLISECONDS);

// 跟进TimeoutCheckTask任务的run方法
private static class TimeoutCheckTask implements TimerTask {

    private final Long requestID;

    TimeoutCheckTask(Long requestID) {
        this.requestID = requestID;
    }

    @Override
    public void run(Timeout timeout) {
        DefaultFuture future = DefaultFuture.getFuture(requestID);
        // 如果任务的deadline到了，那么检测future的结果
        if (future == null || future.isDone()) {
            return;
        }
        // future未完成，设置超时状态
        if (future.getExecutor() != null) {
            future.getExecutor().execute(() -> notifyTimeout(future));
        } else {
            notifyTimeout(future);
        }
    }

    private void notifyTimeout(DefaultFuture future) {
        // create exception response.
        Response timeoutResponse = new Response(future.getId());
        // 设置超时状态，如果消息没发送，客户端超时，反之服务端超时
        timeoutResponse.setStatus(future.isSent() ? Response.SERVER_TIMEOUT : Response.CLIENT_TIMEOUT);
        timeoutResponse.setErrorMessage(future.getTimeoutMessage(true));
        // 处理response，并且设置 CompletableFuture的执行异常
        DefaultFuture.received(future.getChannel(), timeoutResponse, true);
    }
}

// 跟进 DefaultFuture.received
public static void received(Channel channel, Response response, boolean timeout) {
    try {
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
            Timeout t = future.timeoutCheckTask;
            if (!timeout) {
                // decrease Time
                t.cancel();
            }
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
// 跟进future.doReceived
private void doReceived(Response res) {
    if (res == null) {
        throw new IllegalStateException("response cannot be null");
    }
    if (res.getStatus() == Response.OK) {
        this.complete(res.getResult());
    } 
    // 检测 res的超时状态
    else if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
        // 如果发生超时，那么给当前 CompletableFuture 设置异常信息，并且是CompletionException包了TimeoutException。注意：DefaultFuture extends CompletableFuture<Object>
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

// 上述在response设置了超时状态，那么怎么样触发超时异常的呢？

注意：先执行 AsyncToSyncInvoker ，然后在AsyncToSyncInvoker方法内执行AbstractInvoker

//org.apache.dubbo.rpc.protocol.AsyncToSyncInvoker#invoke
public Result invoke(Invocation invocation) throws RpcException {
    Result asyncResult = invoker.invoke(invocation);

    try {
        // 如果是同步调用
        if (InvokeMode.SYNC == ((RpcInvocation) invocation).getInvokeMode()) {
            // 虽然设置的是 Integer的最大值，但是另有玄机
            asyncResult.get(Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
        }
    } catch (InterruptedException e) {
        throw new RpcException("Interrupted unexpectedly while waiting for remote result to return!  method: " +
                invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (ExecutionException e) {
        // 最终会在这里捕获超时异常
        Throwable t = e.getCause();
        if (t instanceof TimeoutException) {
            // 如果方法调用超时了，可以搜到这行日志
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " +
                    invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } else if (t instanceof RemotingException) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " +
                    invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } else {
            throw new RpcException(RpcException.UNKNOWN_EXCEPTION, "Fail to invoke remote method: " +
                    invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    } catch (Throwable e) {
        throw new RpcException(e.getMessage(), e);
    }
    return asyncResult;
}

// 跟进asyncResult.get
// org.apache.dubbo.rpc.AsyncRpcResult#get(long, java.util.concurrent.TimeUnit)
 @Override
public Result get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
    if (executor != null && executor instanceof ThreadlessExecutor) {
        ThreadlessExecutor threadlessExecutor = (ThreadlessExecutor) executor;
        threadlessExecutor.waitAndDrain();
    }
    return responseFuture.get(timeout, unit);
}

// 最终会调用 CompletableFuture 的get方法
// java.util.concurrent.CompletableFuture#reportGet
private static <T> T reportGet(Object r)
    throws InterruptedException, ExecutionException {
    if (r == null) // by convention below, null means interrupted
        throw new InterruptedException();
    if (r instanceof AltResult) {
        Throwable x, cause;
        // 获取CompletableFuture的异常
        if ((x = ((AltResult)r).ex) == null)
            return null;
        if (x instanceof CancellationException)
            throw (CancellationException)x;
        if ((x instanceof CompletionException) &&
            (cause = x.getCause()) != null)
            // 此时会满足这个条件，进而抛出ExecutionException，带上原始的Exception
            x = cause;
        throw new ExecutionException(x);
    }
    @SuppressWarnings("unchecked") T t = (T) r;
    return t;
}
```


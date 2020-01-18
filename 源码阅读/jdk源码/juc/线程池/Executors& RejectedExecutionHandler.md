## Executors静态方法与拒绝策略分析：
### 静态方法创建线程池：
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    // 定长线程池，corePoolSize==maxPoolSize. 线程空闲时间为0
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
}

public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}

public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
// 还有可调度的线程池
```

### 拒绝策略：
```java
// 默认拒绝策略，不处理任务(丢弃任务)，然后抛出异常
public static class AbortPolicy implements RejectedExecutionHandler {
    public AbortPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                                " rejected from " +
                                                e.toString());
    }
}

// 不处理任务。啥也不干
public static class DiscardPolicy implements RejectedExecutionHandler {
    public DiscardPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}

// 直接丢弃最早在队列的任务(也就是等待时间最长的任务)，然后把当前任务加入队列
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    public DiscardOldestPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        // 线程池没关闭
        if (!e.isShutdown()) {
            // FIFO，弹出队首元素
            e.getQueue().poll();
            // 把当前任务加入队列
            e.execute(r);
        }
    }
}

// 只要线程池没有被关闭，那么由提交任务的线程自己来执行这个任务。
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public CallerRunsPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            // 线程同步执行Runnable的run方法
            r.run();
        }
    }
}
```


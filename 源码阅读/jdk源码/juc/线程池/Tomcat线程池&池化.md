# 池化技术
## CPU密集型
首先， JDK 实现的这个线程池优先把任务放入队列暂存起来，而不是创建更多的线程，它比较适用于执行 CPU 密集型的任务，也就是需要执行大量 CPU 运算的任务。这是为什么呢？因为执行 CPU 密集型的任务时 CPU 比较繁忙，因此只需要创建和 CPU 核数相当的线程就好了，多了反而会造成线程上下文切换，降低任务执行效率。所以当前线程数超过核心线程数时，线程池不会增加线程，而是放在队列里等待核心线程空闲下来。

## IO密集型
但是，我们平时开发的 Web 系统通常都有大量的 IO 操作，比方说查询数据库、查询缓存等等。任务在执行 IO 操作的时候 CPU 就空闲了下来，这时如果增加执行任务的线程数而不是把任务暂存在队列中，就可以在单位时间内执行更多的任务，大大提高了任务执行的吞吐量。```所以你看 Tomcat 使用的线程池就不是 JDK 原生的线程池，而是做了一些改造，当线程数超过 coreThreadCount 之后会优先创建线程，直到线程数到达 maxThreadCount，这样就比较适合于 Web 系统大量 IO 操作的场景了，你在实际使用过程中也可以参考借鉴。```其次，线程池中使用的队列的堆积量也是我们需要监控的重要指标，对于实时性要求比较高的任务来说，这个指标尤为关键。

最后，如果你使用线程池请一定记住不要使用无界队列（即没有设置固定大小的队列）。也许你会觉得使用了无界队列后，任务就永远不会被丢弃，只要任务对实时性要求不高，反正早晚有消费完的一天。但是，大量的任务堆积会占用大量的内存空间，一旦内存空间被占满就会频繁地触发 Full GC，造成服务不可用，我之前排查过的一次 GC 引起的宕机，起因就是系统中的一个线程池使用了无界队列。

## 需要注意的点
- 池子的最大值和最小值的设置很重要，初期可以依据经验来设置，后面还是需要根据实际运行情况做调整。
- 池子中的对象需要在使用之前预先初始化完成，这叫做池子的预热，比方说使用线程池时就需要预先初始化所有的核心线程。如果池子未经过预热可能会导致系统重启后产生比较多的慢请求。
- 池化技术核心是一种空间换时间优化方法的实践(缓存亦是如此)，所以要关注空间占用情况，避免出现空间过度使用出现内存泄露或者频繁垃圾回收等问题。

# Tomcat线程池源码分析
```java
// 默认的队列值大小
protected int maxQueueSize = Integer.MAX_VALUE;

// org.apache.catalina.core.StandardThreadExecutor#startInternal
@Override
protected void startInternal() throws LifecycleException {
    // 可以自定义 maxQueueSize
    taskqueue = new TaskQueue(maxQueueSize);
    TaskThreadFactory tf = new TaskThreadFactory(namePrefix,daemon,getThreadPriority());
    executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), maxIdleTime, TimeUnit.MILLISECONDS,taskqueue, tf);
    executor.setThreadRenewalDelay(threadRenewalDelay);
    if (prestartminSpareThreads) {
        executor.prestartAllCoreThreads();
    }
    taskqueue.setParent(executor);

    setState(LifecycleState.STARTING);
}

// org.apache.tomcat.util.threads.ThreadPoolExecutor#ThreadPoolExecutor(int, int, long, java.util.concurrent.TimeUnit, java.util.concurrent.BlockingQueue<java.lang.Runnable>, java.util.concurrent.ThreadFactory)
// 构造函数
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory) {
    super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, new RejectHandler());
    // 预热，启动所有coreThreads
    prestartAllCoreThreads();
}

// 先看下execute 方法
// org.apache.tomcat.util.threads.ThreadPoolExecutor#execute(java.lang.Runnable, long, java.util.concurrent.TimeUnit)
public void execute(Runnable command, long timeout, TimeUnit unit) {
    // 记录提交的任务数
    submittedCount.incrementAndGet();
    try {
        // 调用 jdk的 execute 方法
        super.execute(command);
    } catch (RejectedExecutionException rx) {
        // 捕获 拒绝异常
        if (super.getQueue() instanceof TaskQueue) {
            final TaskQueue queue = (TaskQueue)super.getQueue();
            try {
                if (!queue.force(command, timeout, unit)) {
                    submittedCount.decrementAndGet();
                    throw new RejectedExecutionException(sm.getString("threadPoolExecutor.queueFull"));
                }
            } catch (InterruptedException x) {
                submittedCount.decrementAndGet();
                throw new RejectedExecutionException(x);
            }
        } else {
            submittedCount.decrementAndGet();
            throw rx;
        }

    }
}

// 重点看
// org.apache.tomcat.util.threads.TaskQueue#offer

public class TaskQueue extends LinkedBlockingQueue<Runnable> {

  ...
   @Override
  //线程池调用任务队列的方法时，当前线程数(Worker数量)肯定已经大于核心线程数了
  public boolean offer(Runnable o) {

      //如果线程数已经到了最大值，不能创建新线程了，只能把任务添加到任务队列。
      if (parent.getPoolSize() == parent.getMaximumPoolSize()) 
          return super.offer(o);
          
      //执行到这里，表明当前线程数大于核心线程数，并且小于最大线程数。
      //表明是可以创建新线程的，那到底要不要创建呢？分两种情况：
      
      //1. 如果已提交的任务数小于当前线程数，表示还有空闲线程，无需创建新线程
      if (parent.getSubmittedCount()<=(parent.getPoolSize())) 
          return super.offer(o);
          
      //2. 如果已提交的任务数大于当前线程数，线程不够用了，返回false去创建新线程
      if (parent.getPoolSize()<parent.getMaximumPoolSize()) 
          return false;
          
      //默认情况下总是把任务添加到任务队列
      return super.offer(o);
  }
  
}
```

# 总结
池化的目的是为了避免频繁地创建和销毁对象，减少对系统资源的消耗。Java 提供了默认的线程池实现，我们也可以扩展 Java 原生的线程池来实现定制自己的线程池，Tomcat 就是这么做的。Tomcat 扩展了 Java 线程池的核心类 ThreadPoolExecutor，并重写了它的 execute 方法，定制了自己的任务处理流程。```同时 Tomcat 还实现了定制版的任务队列，重写了 offer 方法，使得在任务队列长度无限制的情况下，线程池仍然有机会创建新的线程。```
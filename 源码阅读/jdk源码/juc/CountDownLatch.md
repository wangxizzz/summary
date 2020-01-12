## 真正的应用场景：
等待分析...

## 源码分析:

### 构造函数分析：
```java
// 构造函数
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

Sync(int count) {
    // 设置AQS的state值
    setState(count);
}
```
### countDown()分析：
```java
public void countDown() {
    sync.releaseShared(1);
}
// AQS共享锁释放方法
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        // 对于CountDownLatch来说，tryReleaseShared返回true意味着,state==0
        // 所以此时应该唤醒被await方法阻塞的线程(一个方法只有主线程)，因此此处只需要唤醒一次。
        doReleaseShared();
        return true;
    }
    return false;
}
// 子类重写的尝试释放方法
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    // 自旋把state减一，当state为0时，返回true给上层.
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

### await()分析：
**```方法说明：不带参数的await方法，会一直等待所有子线程生成自己的运行结果。也就是说await方法会等待到耗时最长的子任务完成。(注意：线程池提交的都是异步任务，不论是execute还是submit方法提交)```**
```java
// 此方法会抛出中断异常，但是没有超时异常这一说！
public void await() throws InterruptedException {
    // 调用AQS
    sync.acquireSharedInterruptibly(1);
}
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        // 进入这里，说明至少剩余一个子线程的countDown方法没有调用
        doAcquireSharedInterruptibly(arg);
}
// 子类重写的
protected int tryAcquireShared(int acquires) {
    // 如果state==0，说明子线程的countDown方法都调用完毕，那么此时就不用阻塞主线程了。
    return (getState() == 0) ? 1 : -1;
}

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 节点封装，进入AQS阻塞队列
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            // 获取节点的前驱节点
            final Node p = node.predecessor();
            if (p == head) {
                // 子类重写的tryAcquireShared
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 进入这里，说明子任务没完成，那么挂起当前线程。这就是所谓的await方法阻塞主线程等待多个子线程完成的代码
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### await(long timeout, TimeUnit unit)分析
**```此方法是应用于防止main线程一直阻塞。方法分析：带有超时时间的await方法，只要过了这个时间，不论子任务执行完毕与否，那么该方法的主线程都会从await方法恢复，继续往下执行，而且此方法捕获的是可中断异常，并不会抛出超时异常。因此，超时时间的设置决定了主线程往下执行的正确性。```**

**那为什么过了时间，主线程就自己往下走了呢？**
```java
// 注意：这个方法时主线程(相对于子线程)调用的
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
// 调用AQS方法
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
        doAcquireSharedNanos(arg, nanosTimeout);
}
// 这个是await(long timeout, TimeUnit unit)真正核心所在
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                // 当子任务没完成，这里返回-1
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            // 能走到这里来，说明还有子任务没完成，那么判断是否已超时
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                // 超时就直接返回，不会阻塞主线程
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                // 如果没超时，那么会继续等待剩余时间nanosTimeout，然后自己返回
                // 超时返回重点在这里
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

关于LockSupport的demo:
参见j2se工程：多线程.java并发编程之美.ch6锁原理剖析.LockSupport02
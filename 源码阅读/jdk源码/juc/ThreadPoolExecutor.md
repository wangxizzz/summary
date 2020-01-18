## 介绍：
线程池主要解决两个问题：  
（1）当执行大量异步任务时线程池可以提供更好的性能。在不使用线程池时，每次执行异步任务时，都要new一个线程来运行，而线程的创建于销毁是有开销的。线程池里的线程是可以复用的，不需要每次执行异步任务时创建于销毁线程。  
（2）线程池提供了一种资源管理与限制的手段，比如可以动态的增加与减少线程的个数，还提供了一些统计信息，比如当前线程池完成任务的数目。

## 源码分析
### 成员变量介绍
```java
// ctl原子变量，是用来记录线程池的状态与线程池中线程的个数。类似于ReentrantReadWriteLock使用一个变量保存两种信息。32位的高3位用来表示线程池的状态，低29位表示线程的个数。
// 默认是Running状态，线程个数为0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 线程个数掩码
private static final int COUNT_BITS = Integer.SIZE - 3;
// 线程池中线程的最大数目 000111...(29个1)
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// 线程池的运行状态存储在ctl的高三位

// 高三位：111
private static final int RUNNING    = -1 << COUNT_BITS;
// 000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 001
private static final int STOP       =  1 << COUNT_BITS;
// 010
private static final int TIDYING    =  2 << COUNT_BITS;
// 011
private static final int TERMINATED =  3 << COUNT_BITS;

// ~CAPACITY  取 非运算
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 两个变量做 按位 或运算。就可以记录两种状态
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

### 状态变量的介绍：
RUNNING：接受新的任务，处理等待队列中的任务  
SHUTDOWN：不接受新的任务提交，但是会继续处理等待队列中的任务  
STOP：不接受新的任务提交，不再处理等待队列中的任务，中断正在执行任务的线程  
TIDYING：所有的任务都销毁了，workCount 为 0。线程池的状态在转换为 TIDYING 状态时，会执行钩子方法 terminated()  
TERMINATED：terminated() 方法结束后，线程池的状态就会变成这个  

### 状态变量的转换：
RUNNING -> SHUTDOWN：当调用了 shutdown() 后，会发生这个状态转换，这也是最重要的  
(RUNNING or SHUTDOWN) -> STOP：当调用 shutdownNow() 后，会发生这个状态转换，这下要清楚 shutDown() 和 shutDownNow() 的区别了  
SHUTDOWN -> TIDYING：当任务队列和线程池都清空后，会由 SHUTDOWN 转换为 TIDYING  
STOP -> TIDYING：当任务队列清空后，发生这个转换  
TIDYING -> TERMINATED：这个前面说了，当 terminated() 方法结束后  

### execute(Runnable command)介绍
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // 如果工作线程小于核心线程数
    if (workerCountOf(c) < corePoolSize) {
        // 添加一个线程
        if (addWorker(command, true))
            return;
        // 如果添加失败，那么重新获取状态值
        c = ctl.get();
    }
    // 走到这里，说明 workerCountOf(c) >= corePoolSize
    if (isRunning(c) && workQueue.offer(command)) {
        // 判断线程池状态，并且offer一个任务到阻塞队列。offer方法当队列满时会返回false.

        // 重新请求状态值(因为有可能此时调用了shutDown方法)
        int recheck = ctl.get();
        // 如果线程池不是Running状态，那么需要移除加入的任务
        if (! isRunning(recheck) && remove(command))
            // 移除成功，采用拒绝策略。
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 走到这里，说明队列满了，那么增加Worker到maxThread
    else if (!addWorker(command, false))
        reject(command);
}
```

### Worker成员变量介绍：
```java
// Worker本身是一个Runnable
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
{
    // 执行任务的线程
    final Thread thread;
    
    Runnable firstTask;
    // 次Worker完成任务的数量总数
    volatile long completedTasks;

    // 构造函数
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // 利用线程工厂创建线程，并传入当前Worker对象(也就是传入一个Runnable)
        this.thread = getThreadFactory().newThread(this);
    }
    // 重写Runnable接口的方法
    @Override
    public void run() {
        // 真正执行外界任务的方法
        runWorker(this);
    }
```
> 注意：上面的Worker构造函数，把AQS中的state变量设置为 -1 ，这是为了避免当前Worker在调用runWorker方法前被中断，因为当其他线程都用shutDownNow方法时，会中断state >= 0的线程，所以这里设置为-1，就不会被中断了。在runWorker方法调用worker.unlock会把state设置为0，允许中断。

### addWorker方法分析：
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 检查线程池的状态
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
            return false;

        for (;;) {
            // 获取工作线程数
            int wc = workerCountOf(c);
            // 判断线程数是否到达最大容量。
            if (wc >= CAPACITY ||
                // 判断是否大于corePoolSize，大于就addWorker失败，加入阻塞队列。
                // 判断是否大于最大线程数，大于就上层执行拒绝策略
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // CAS给Worker数加1
            if (compareAndIncrementWorkerCount(c))
                // 加1成功，直接跳出二层循环，进入到下面的代码1
                break retry;
            // 走到这里，说明CAS失败
            c = ctl.get();  // Re-read ctl
            // 判断线程池状态是否改变，如果改变，直接跳出内存循环，重新外层循环，获取线程池大小
            if (runStateOf(c) != rs)
                // 相当于break操作。
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    // 代码1
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 创建一个Worker
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 这里加锁，保证多线程下workers.add成功
            mainLock.lock();
            try {
                // 重新获取线程池状态，防止加锁过程中，有线程调用了shutDown
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 增加Worker数目
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    // worker成功增加，更新标志变量，启动Worker线程执行任务
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                // 启动Worker线程
                // 调用Worker的成员变量Thread.start方法，本质是调用Worker(Runnable)的run方法，也就是java.util.concurrent.ThreadPoolExecutor#runWorker
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            // 新增加Worker失败的处理
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

增加Worker失败的处理:
```java
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        decrementWorkerCount();
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```
步骤如下：  
1.加锁，因为workers是HashSet结构；  
2.移除Set里的worker(销毁Worker对象的成员变量thread)，如果存在；  
3.worker数量减一;  

### runWorker方法分析：
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 把state变量设置为0,允许中断
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // while循环去处理任务，在这里体现出线程池的线程复用策略
        while (task != null || (task = getTask()) != null) {
            // 设置state变量为1
            w.lock();
            // 判断状态。
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 任务执行前的处理
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 外界任务的真正执行
                    task.run();
                } catch (RuntimeException x) {
                    // 抛出外界任务执行时，抛出的异常。
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 任务执行后的处理
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                // 统计每个worker完成任务的数目
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // worker退出的处理。能走到这里，说明
        // 1.发生异常，并且由线程池抛出
        // 2.队列里的任务处理完了，获取task为null,线程有idle的状态,应该回收线程
        processWorkerExit(w, completedAbruptly);
    }
}
```
### 线程池状态的方法：
```java
private static boolean runStateLessThan(int c, int s) {
     return c < s;
}

private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}

private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
```
```java
/**
* If false (default), core threads stay alive even when idle.
* If true, core threads use keepAliveTime to time out waiting
* for work.
*/
// 控制核心线程是否消亡的变量
private volatile boolean allowCoreThreadTimeOut;
// 通过调用此方法设置allowCoreThreadTimeOut的值，默认为false，如果设置为true，当核心线程空闲时，会尝试回收这些线程。
public void allowCoreThreadTimeOut(boolean value) {
    if (value && keepAliveTime <= 0)
        throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
    if (value != allowCoreThreadTimeOut) {
        allowCoreThreadTimeOut = value;
        if (value)
            interruptIdleWorkers();
    }
}
```
**关于allowCoreThreadTimeOut变量：是否回收核心线程**  
```keepAliveTime表示空闲线程的保活时间，如果某线程的空闲时间超过这个值都没有任务给它做，那么可以被关闭了。注意这个值并不会对所有线程起作用，如果线程池中的线程数少于等于核心线程数 corePoolSize，那么这些线程不会因为空闲太长时间而被关闭，当然，也可以通过调用 allowCoreThreadTimeOut(true)使核心线程数内的线程也可以被回收。```

### getTask()分析：从工作队列中获取外界提交的任务
```java
private Runnable getTask() {
    boolean timedOut = false; 
    // 死循环
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);
        // 如果允许【核心】线程消亡或者 worker线程数大于corePoolSize,timed=true
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            // 如果复合线程消亡的条件，那么就回收一个worker线程(线程池的收缩在这里体现)
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            // 如果获取的任务为null，设置timedOut = true, 说明有线程是idle状态，继续for循环，即可回收idle现线程
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

### processWorkerExit方法的分析：处理Worker线程回收
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果上层抛出了异常，那么completedAbruptly=true，那么直接降低Worker的数量
    if (completedAbruptly) 
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 把该worker完成的任务累加到总任务上
        completedTaskCount += w.completedTasks;
        // 移除这个Worker，内部把worker对象置为null,被GC回收，回收了worker成员变量thread.
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    // 尝试终止线程池
    tryTerminate();

    int c = ctl.get();
    // 如果线程池时SHUTDOWN、RUNNING状态
    if (runStateLessThan(c, STOP)) {
        // 执行任务没发生异常
        if (!completedAbruptly) {
            // 是否允许回收核心线程。默认是false
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        // 如果线程数小于 min，那么增加一个Worker.
        addWorker(null, false);
    }
}
```
## 线程池关闭分析：
### shutdown方法分析：
```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 检查权限
        checkShutdownAccess();
        // 设置当前线程池状态为SHUTDOWN,如果是SHUTDOWN，那么直接返回
        advanceRunState(SHUTDOWN);
        // 设置中断标志
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    // 尝试将状态设置为Terminate状态
    tryTerminate();
}
```
```调用shutDown()方法后，线程池就不会再接收新任务了(支撑这个的语义是在执行execute方法时，判断线程池的状态来决定是否执行addWorker和workQueue.offer(command)方法), 但是此时线程池仍然会处理阻塞队列里面的任务。shutDown方法会立即返回，并不会等待队列里的任务执行完成。```

```advanceRunState```源码如下
```java
private void advanceRunState(int targetState) {
    for (;;) {
        int c = ctl.get();
        // 如果当前状态等于或者大于targetState，那么跳出循环返回
        if (runStateAtLeast(c, targetState) ||
            // 如果前一个条件不满足，那么CAS设置为targetState
            ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
            break;
    }
}
```
```中断空闲线程interruptIdleWorkers分析：```
```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    // 获取独占锁，保证workers遍历的线程安全性
    mainLock.lock();
    try {
        // 获取所有worker线程
        for (Worker w : workers) {
            Thread t = w.thread;
            // 如果worker线程没有被中断并且 内部锁尝试获取成功(设置state=1)。
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    // 那么设置中断
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    // 释放内部锁，设置state=0
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```
**```能被中断线程如下：```**  
1.因为前提是要获取worker内部锁，因此在runWorker时，也就是任务正在执行时，不会被中断，因为此时内部锁已经加了。  
2.刚创建的Worker线程，不会被中断，因为刚创建的Worker的state=-1，不是expect=0。  
3.**这里可以中断是，调用getTask方法，阻塞到获取阻塞队列任务的take方法上，也就是空闲线程**。

### shutdownNow方法分析：
```java
// 返回阻塞队列的任务列表
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 设置线程池状态为STOP
        advanceRunState(STOP);
        // 中断所有线程
        interruptWorkers();
        // 把阻塞队列的任务返回
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```
**调用shutDownNow方法会中断所有线程：**
```java
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 中断所有线程
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}

void interruptIfStarted() {
    Thread t;
    // 当worker刚创建时，state=-1，因此中断所有正在运行的Worker线程，其本质时中断所有Worker线程，因为worker创建完毕，就会立即调用runWorker方法。
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
```

## 总结：
至此：线程池核心源码分析完毕。从执行execute() -> addWorker -> runWorker -> getTask -> 任务处理 -> 处理回收空闲线程processWorkerExit -> 线程池关闭shutDown或shutDownNow.

线程池非常巧妙的利用了一个Integer类型的原子变量ctl，来记录线程池的状态与线程个数。通过线程池的状态来控制任务的运行，每个worker可以处理多个任务。线程池可以通过线程的复用减少线程创建于销毁的开销。
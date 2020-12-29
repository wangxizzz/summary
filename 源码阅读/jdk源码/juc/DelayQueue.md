# 描述
DelayQueue 是一个支持延时获取元素的无界阻塞队列。队列使用 ```PriorityQueue``` 来实现。 队列中的元素必须实现 Delayed 接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。

# DelayQueue 使用场景

## DelayQueue 特点
DelayQueue 也是一种比较特殊的阻塞队列，从类声明也可以看出，DelayQueue 中的所有元素必须实现 Delayed 接口。DelayQueue 队列的元素必须实现 Delayed 接口。

```java
// 此接口的实现必须定义一个 compareTo 方法，该方法提供与此接口的 getDelay 方法一致的排序。
public interface Delayed extends Comparable<Delayed> {
    // 返回与此对象相关的剩余有效时间，以给定的时间单位表示
    long getDelay(TimeUnit unit);
}
```
可以看到，Delayed 接口除了自身的 getDelay 方法外，还实现了 Comparable 接口。getDelay 方法用于返回对象的剩余有效时间，实现 Comparable 接口则是为了能够比较两个对象，以便排序。

也就是说，如果一个类实现了 Delayed 接口，当创建该类的对象并添加到 DelayQueue 中后，只有当该对象的 getDalay 方法返回的剩余时间 ≤0 时才会出队。

另外，由于 DelayQueue 内部委托了 PriorityQueue 对象来实现所有方法，所以能以堆的结构维护元素顺序，这样剩余时间最小的元素就在堆顶，每次出队其实就是删除剩余时间 ≤0 的最小元素。

DelayQueue 的特点简要概括如下：
- DelayQueue 是无界阻塞队列；
- 队列中的元素必须实现 Delayed 接口，元素过期后才会从队列中取走；

## DelayQueue 使用场景
DelayQueue 非常有用，可以将 DelayQueue 运用在以下应用场景。
1. 缓存系统的设计：可以用 DelayQueue 保存缓存元素的有效期，使用一个线程循环查询 DelayQueue，一旦能从 DelayQueue 中获取元素时，表示缓存有效期到了。
2. 定时任务调度：使用 DelayQueue 保存当天将会执行的任务和执行时间，一旦从 DelayQueue 中获取到任务就开始执行，比如 javax.swing.TimerQueue 就是使用 DelayQueue 实现的。ScheduledFutureTask

# DelayQueue 源码分析
```java
// 同步
private final transient ReentrantLock lock = new ReentrantLock();
private final Condition available = lock.newCondition();

// PriorityQueue 维护队列
private final PriorityQueue<E> q = new PriorityQueue<E>();
private Thread leader = null;
```

## 元素入队列
```java
public boolean offer(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        q.offer(e);             // 调用 PriorityQueue#offer 方法
        if (q.peek() == e) {    // 如果入队元素在队首, 则唤醒一个出队线程
            leader = null;
            available.signal();
        }
        return true;
    } finally {
        lock.unlock();
    }
}
```

## 元素出队列，阻塞take
```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            // 1. 集合为空时所有的线程都处于无限等待的状态。
            //    只要有元素将其中一个线程转为 leader 状态
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                // 2. 元素已经过期，直接取出返回(当前时间超过deadline时间任务就会被执行)
                if (delay <= 0)
                    return q.poll();
                first = null; // don't retain ref while waiting
                // 3. 已经在其它线程设置为 leader，无限期等着
                if (leader != null)
                    available.await();
                // 4. 将 leader 设置为当前线程，阻塞当前线程（限时等待剩余有效时间）
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        // 等待一定时间超时，继续循环
                        available.awaitNanos(delay);
                    } finally {
                        // 4.1 尝试获取过期的元素，重新竞争
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        // 5. 队列中有元素则唤醒其它无限等待的线程
        //    leader 线程是限期等待，每次 leader 线程获取元素出队，如果队列中有元素
        //    就要唤醒一个无限等待的线程，将其设置为限期等待，也就是总有一个等待线程是 leader 状态
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```

# delayQueue带来的问题
消费者线程的数量要够，处理任务的速度要快。否则，队列中的到期元素无法被及时取出并处理，造成任务延期、队列元素堆积等情况。  
具体问题可见  j2se工程的 多线程.Java并发编程.day8延迟队列.example01.DelayQueueExample
## 介绍：
AQS全称是 AbstractQueuedSynchronizer （抽象队列同步器），是通过一个先进先出的队列（存储等待的线程）来实现同步器的一个框架是一个抽象类，是java.util.concurrent包下很多多线程工具类的实现基础。Lock、CountDownLatch、Semaphore等都是基于AQS实现的。AQS是一个FIFO的双向链表结构，其内部通过head、tail节点标识链表的首节点与尾结点，队列元素为Node类型，Node中的Thread变量是是用来存放到AQS队列里面的线程。

## AQS的实现：
AQS支持两种锁一种是独占锁（独占模式），一种是共享锁（共享模式）
- 独占锁：比如像ReentrantLock就是一种独占锁模式，多个线程去同时抢一个锁，只有一个线程能抢到这个锁，其他线程就只能阻塞等待锁被释放后重新竞争锁。
- 共享锁：比如像读写锁里面的读锁，一个锁可以同时被多个线程拥有（多个线程可以同时拥有读锁），再比如Semaphore 设置一个资源数目（可以理解为一个锁能同时被多少个线程拥有）。
- 共享锁跟独占锁可以同时存在，比如比如读写锁，读锁写锁分别对应共享锁和独占锁。

## 源码分析：
### AQS成员变量：
```java
//AQS等待队列的头结点，AQS的等待队列是基于一个双向链表来实现的，这个头结点并不包含具体的线程是一个空结点（注意不是null）
private transient volatile Node head;
//AQS等待队列的尾部结点
private transient volatile Node tail;
//AQS同步器状态，也可以说是锁的状态，注意volatile修饰证明这个变量状态要对多线程可见
private volatile int state;
```
>> AQS内部维持了一个单一状态信息state，可以通过```getState、setState、compareAndSetState```修改state的值。在ReentrantLock里，state表示当前线程获得锁的可重入次数。对于读写锁ReentrantReadWriteLock来说，state的高16位表示读状态，也就是获取读锁的次数，低16位表示该线程获取写锁的可重入次数。对于Semaphore来说，state表示当前可用信号的个数；对于CountDownLatch来说，state表示计数器当前的值。对于AQS来说，线程同步的关键是对state变量的操作，根据state是否属于一个线程，操作的方式分为独占和共享。

### AQS内部类Node:
Node顾名思义就是接点的意思，前面说过AQS等待队列是一个双链表，每个线程进入AQS的等待队列的时候都会被包装成一个Node节点。
```java
static final class Node {
    // 共享锁是空对象
    static final Node SHARED = new Node();
    // 独占锁是null
    static final Node EXCLUSIVE = null;

    // 下面这四个属性就是说明结点(当前线程的状态)的状态
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
    volatile int waitStatus;
    // 前驱节点
    volatile Node prev;
    // 后继节点
    volatile Node next;
    // 结点所包装的线程
    volatile Thread thread;
    //对于Condtion表示下一个等待条件变量的节点；其它情况下用于区分共享模式和独占模式
    Node nextWaiter;
  }
```
>> Node节点内部的SHARED是用来标记该线程在获取共享资源被阻塞挂起后放入AQS队列的；EXCLUSIVE用来标记该现场获取独占资源阻塞挂起后放入AQS队列的。waitStatus记录当前线程的状态，可以为CANCELLED(线程被取消了),SIGNAL(线程需要被唤醒),CONDITION(线程在条件队列中等待),PROPAGATE(释放共享资源时，需要通知其他节点)。

## 具体分析
### 可重入锁ReentranLock 分析：
```java
public ReentrantLock() {
    // 默认非公平锁
    sync = new NonfairSync();
}
public void lock() {
    sync.lock();
}
```
#### 非公平锁分析：
```java
// 非公平锁分析：
final void lock() {
    // 如果锁没有被其他线程占用并且当前线程当前线程之前没有获取到该锁
    if (compareAndSetState(0, 1))
        // CAS成功，表示当前线程获得锁，state设置为1，设置当前锁的拥有者为当前线程。
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 调用AQS acquire
        acquire(1);
}
// CAS设置state的值
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
// 设置当前锁拥有者是当前线程
protected final void setExclusiveOwnerThread(Thread thread) {
    exclusiveOwnerThread = thread;
}
// java.util.concurrent.locks.AbstractQueuedSynchronizer#acquire
// 独占模式下获取锁的基本方法
public final void acquire(int arg) {
    // 如果尝试获取锁失败tryAcquire返回false, 那么加入AQS队列
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
// java.util.concurrent.locks.ReentrantLock.NonfairSync#tryAcquire
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 获取AQS的成员变量state的值
    int c = getState();
    // 当前AQS的state=0
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果是当前线程正好是锁的持有者(可重入锁)
    else if (current == getExclusiveOwnerThread()) {
        // 获取可重入次数。(注意：int溢出的处理方式)
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 获取锁失败 返回false
    return false;
}
```
#### AQS 独占模式下获取锁的基本方法acquire(int arg)分析：
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
>> 该方法会试图获取锁，如果获取不到，就会被加入等待队列等待被唤醒.首先是 tryAcquire 方法，也就是尝试获取锁，该方法是需要被写的，父类默认的方法是抛出异常。如何重写呢？抽象类定义一个标准：如果返回 true，表示获取锁成功，反之失败。在ReenterLock已经分析。我们回到 acquire 方法，如果获取锁成功，就直接返回了，如果失败了，则继续后面的操作，也就是将线程放入等待队列中：

- tryAcquire方法抢锁失败就执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
- 先是addWaiter(Node.EXCLUSIVE)方法，这个方法表示将当前抢锁线程包装成结点并加入等待队列，返回包装后的结点
- addWaiter方法返回的结点，作为acquireQueued方法的参数，该方法主要是等待队列顺序获取资源
- 注意acquireQueued返回true表示线程发生中断，这时就会执行selfInterrupt方法响应中断。
由于tryAcquire AQS没有具体实现，下面我们就接着看下addWaiter这个方法:
```java
// 将当前线程放入到队列节点
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}
addWaiter步骤如下：
1.创建一个当前线程的 Node 对象（nextWaiter 属性为 null， thread 属性为 当前线程）
2.如果当前节点node不为null，并且

```

```java

```

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

#### 公平锁分析：
```java

```

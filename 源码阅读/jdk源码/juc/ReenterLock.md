## 可重入锁ReentranLock 分析：
```java
public ReentrantLock() {
    // 默认非公平锁
    sync = new NonfairSync();
}
public void lock() {
    sync.lock();
}
```
### 非公平锁分析lock：
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
    // 当前AQS的state=0,说明没有线程获得锁
    if (c == 0) {
        // 利用CAS 把state置为acquires
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

### ReentrantLock 公平锁Lock方法分析：
```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
// 公平锁的tryAcquire
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // hasQueuedPredecessors 这里控制锁的公平获取
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            // CAS 设置state成功
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 可重入锁
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
>> hasQueuedPredecessors控制线程获取锁的公平性，分析如下：
```java
public final boolean hasQueuedPredecessors() {
    Node t = tail;
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
方法分析如下：  
- 如果h==t，说明AQS队列中没有节点，那么直接返回false，上层方法就进入尝试获取锁的方法。
- 如果队列中有Node，但是head节点的第一个节点Node成员变量thread不是当前线程，那么hasQueuedPredecessors返回true，该线程阻塞；如果是当前线程，那么hasQueuedPredecessors返回false。

### ReentrantLock unlock释放锁分析：
```java
public void unlock() {
    // 调用AQS的release方法
    sync.release(1);
}
```
```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        // 释放锁成功
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 唤醒头结点的下一个节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
尝试释放锁，如果锁释放成功，那么唤醒下一个线程. 看下tryRelease方法()
```java
// java.util.concurrent.locks.ReentrantLock.Sync#tryRelease
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 可重入锁的体现。可重入锁没有释放完，是不会唤醒另一个线程的
    setState(c);
    return free;
}
```
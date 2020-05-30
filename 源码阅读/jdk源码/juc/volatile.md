## 应用场景：
> **适用于单线程写，多线程读的场景。因为volatile不保证原子性。**

### 举例：
AQS的state变量  

java.util.concurrent.locks.ReentrantLock.Sync#nonfairTryAcquire
```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 利用CAS保证只有单个线程写 state变量
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 利用当前线程是否是获取锁的线程保证，单一线程写
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
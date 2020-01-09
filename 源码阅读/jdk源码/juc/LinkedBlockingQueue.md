## 简单介绍：
LinkedBlockingQueue，底层本质是单向链表(带头尾节点),线程安全，由Reenterant独占锁保证。

## 源码分析：
### 成员变量介绍：
```java
// 初始化容量
private final int capacity;
// 当前容器中元素的个数
private final AtomicInteger count = new AtomicInteger();
// 链表头结点(空对象,在构造函数进行初始化)
transient Node<E> head;
// 尾结点
private transient Node<E> last;
// 调用take、poll方法时，保证同时只有一个线程获取
private final ReentrantLock takeLock = new ReentrantLock();

// 当队列为空时，执行出队列操作(take方法)的线程会被放入此条件队列进行等待
private final Condition notEmpty = takeLock.newCondition();
/** 执行put、offer会获取该锁 */
private final ReentrantLock putLock = new ReentrantLock();
// 当队列满时，执行元素入队列操作(put方法)的线程会被阻塞放入次条件队列
private final Condition notFull = putLock.newCondition();
```
### 构造函数
```java
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    // 初始化两个哨兵节点(都是空对象)
    last = head = new Node<E>(null);
}
```
### 插入元素操作  offer、put方法介绍：
```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    // 获取容器中元素的个数
    final AtomicInteger count = this.count;
    // 如果容器满了，直接返回false
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            // 新节点 入队列
            enqueue(node);
            c = count.getAndIncrement();
            // 队列没有满，那么唤醒notFull条件队列里面因为调用了notFull的await操作(比如执行put方法时队列满了)被阻塞的线程
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    // c==0说明，队列中至少有一个元素，因此唤醒消费者线程
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}

// 节点入队列操作
private void enqueue(Node<E> node) {
    last = last.next = node;
}
```
> ```offer方法，当队列满时，不阻塞，直接返回false.```

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {
            // 如果队列满了，那么休眠
            notFull.await();
        }
        enqueue(node);
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}
```
### 取元素操作 poll、take方法介绍
```java
public E poll() {
    final AtomicInteger count = this.count;
    // 如果队列为空，直接返回null
    if (count.get() == 0)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    // 获取取元素的锁
    takeLock.lock();
    try {
        if (count.get() > 0) {
            x = dequeue();
            // getAndDecrement 先取值，再减一
            c = count.getAndDecrement();
            // 唤醒因调用take方法阻塞的线程
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    // c == capacity 说明这个方法结束，队列至少有一个位置空闲的
    if (c == capacity)
        signalNotFull();
    return x;
}
```

```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    // 获取取元素锁
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            // 如果队列中没有元素，直接休眠
            notEmpty.await();
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

> ```ArrayBlockingQueue```只用一个ReentantLock控制读写的线程安全性，是因为ArrayBlockingQueue底层是数组，数组本身就是共享资源，所以需要用一把锁来控制共享资源的访问。
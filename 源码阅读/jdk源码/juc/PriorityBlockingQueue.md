## 介绍
PriorityBlockingQueue是基于堆(完全二叉树)的```无界阻塞队列```,每次出队列，都会返回优先级最高或者最低的元素(会发生堆调整)，因此直接遍历队列元素，不保证有序，默认使用队列元素对象的compareTo方法提供比较规则，也可以自定义排序规则。

> 无界阻塞队列 : 因为PriorityBlockingQueue源码中没有notFull条件变量，因此调用put(E e)就没有阻塞 的逻辑，因此设计成无界队列。
## 源码分析
### 重要成员介绍与构造函数
```java
// 内部真正存储元素的数组
private transient Object[] queue;
// 队列中的元素个数
private transient int size;
// 堆排序的比较器
private transient Comparator<? super E> comparator;
// 独占锁用来控制同时只有一个线程可以入队或者出队操作
private final ReentrantLock lock;
// 当容器元素为空时，调用take，阻塞的条件变量(注意：没有notFull)
private final Condition notEmpty;
// 是一个自旋锁，利用CAS保证同时只有一个线程进行数组扩容。
private transient volatile int allocationSpinLock;

public PriorityBlockingQueue(int initialCapacity,
                                Comparator<? super E> comparator) {
    // 初始化元素
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    this.queue = new Object[initialCapacity];
}
```

### 入队列 offer、put操作源码分析：
```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    // 获取独占锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    // 注意：while循环
    while ((n = size) >= (cap = (array = queue).length))
        // 当size(容器中元素个数)大于等于容器的长度，扩容
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        // 默认比较器为null
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            // 调用实体类实现的比较器
            siftUpUsingComparator(n, e, array, cmp);
        // 容器元素个数加1
        size = n + 1;
        // 唤醒由于 take方法阻塞的线程
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    // 由于是无界队列，所以offer方法会一直返回true
    return true;
}
```
```注意：PriorityBlockingQueue是无界队列，因此put(put方法本质还是调用offer方法)、offer方法都不阻塞，但是调用take队列中没有元素时，会阻塞。```
**继续看扩容的方法：**
```java
private void tryGrow(Object[] array, int oldCap) {
    // 释放掉主线程的锁
    lock.unlock(); // must release and then re-acquire main lock
    Object[] newArray = null;
    // 尝试CAS进行扩容
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                    0, 1)) {
        try {
            // 计算新容量
            int newCap = oldCap + ((oldCap < 64) ?
                                    (oldCap + 2) : // grow faster if small
                                    (oldCap >> 1));
            // 判断新容量溢出
            if (newCap - MAX_ARRAY_SIZE > 0) {
                // 判断旧的容量值，是否已经是MAX_ARRAY_SIZE了
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                // 如果oldCap没有溢出，那么把newCap赋值最大
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)
                // 创建一个新的数组
                newArray = new Object[newCap];
        } finally {
            // 扩容标识置0
            allocationSpinLock = 0;
        }
    }
    // 线程A CAS成功，进入数组扩容逻辑，线程B CAS失败，走到这里，扩容未成功，newArray==null，让出CPU资源。
    if (newArray == null) 
        Thread.yield();
    lock.lock();
    if (newArray != null && queue == array) {
        queue = newArray;
        // 复制元素到新的线程
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```
方法逻辑分析如下：

```JDK设计精妙之处：tryGrow方法，进行数组扩容，方法首先就是释放独占锁lcok，其实这里也可以不释放锁，然后CAS进行数组扩容，但是在扩容期间一直持有锁，扩容是需要时间的，如果扩容期间还占有锁，那么其他线程就不能进行出队与入队操作的，这就大大的降低了并发性。因此，为了提高性能，使用CAS控制只有一个线程进行扩容，并且在扩容前释放掉锁，这样其他线程仍然可以入队与出队操作。```

allocationSpinLock锁CAS失败，就会进入Thread.yield();，让出当前CPU资源，这样的目的是让扩容成功的线程在扩容完成后优先调用lock.lock获得锁，但是并不能保证，调用yeild的线程可能会继续往下走，但是newArray==null，那么获取lock，然后返回上层方法的while循环，又重新进入tryGrow方法，释放独占锁，等待扩容线程成功扩容。```因此在线程扩容时，其他入队的线程通过自旋的方式进行等待扩容线程的完成。```

yield()从未导致线程转到等待/睡眠/阻塞状态。在大多数情况下，yield()将导致线程从运行状态转到可运行状态，但有可能没有效果。

**增加一个元素，如何建堆：**
```java
// 默认比较器为null，会调用容器中对象 T 的比较方法
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    // k代表当前容器中元素的个数(不算新加的元素)，x代表要入队的元素

    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = array[parent];
        if (key.compareTo((T) e) >= 0)
            break;
        array[k] = e;
        k = parent;
    }
    array[k] = key;
}
```
```增加一个元素，是放到数组的末尾，抽象逻辑来说，是完全二叉树的最后一个节点，但是增加元素需要保证堆顶元素最大或最小，所以，会发生堆向上调整。 poll一个元素，则会发生堆向下调整。```

上面是元素进入堆，堆调整的过程代码(JDK的代码写的确实好)。offer进队列，会发生堆调整，并不保证全局元素的有序性，它只保证堆顶(queue[0])元素最大或者最小。因此，依次poll队列中元素，会得到一个有序的序列，因为每次poll方法也会会发生堆调整。  
依次往队列里offer元素：2 4 6 1 => 最终 数组元素依次为：1 2 6 4 (最小堆)

下面来看put()方法：

```java
public void put(E e) {
    offer(e); // never need to block
}
```

### 出队列 poll、take源码分析：
```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 获取独占锁(响应中断)
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null)
            // 队列为空，阻塞
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}
```
元素出队列时，获取独占锁，其他线程是不能进行出队与入队操作，但是扩容线程仍然可以进行。
```java
private E dequeue() {
    // 如果size==1,说明还有一个元素，那么可以把这个元素出队列
    //如果size==0。说明队列中没有元素，返回null
    int n = size - 1;
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        // 返回堆顶元素
        E result = (E) array[0];
        // array[n]代表数组中最后一个元素
        E x = (E) array[n];
        // 把数组中最后一个元素置null
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            // 发生堆调整
            siftDownComparable(0, x, array, n);
        else
            siftDownUsingComparator(0, x, array, n, cmp);
        // size - 1
        size = n;
        return result;
    }
}
```
poll一个元素，返回堆顶元素，会发生堆向下调整，以此保证堆顶元素最大或最小。
```java
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                               int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        // 找到最后一个非叶子节点的下标。找到非叶子节点进行堆调整
        int half = n >>> 1;           
        while (k < half) {
            int child = (k << 1) + 1; 
            Object c = array[child];
            // 获得右子节点
            int right = child + 1;
            // right < n 判断右子节点是否存在
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                // 取左右子节点的最值
                c = array[child = right];
            if (key.compareTo((T) c) <= 0)
                break;
            // 最值调整
            array[k] = c;
            // 子节点递归找最值
            k = child;
        }
        // 把最后一个值安放在合适的位置
        array[k] = key;
    }
}
```
堆向下调整的思路：本质是把最后一个元素向上调整放到合适的位置(因为最后一个元素已经置空了)，然后找到最值放到堆顶，并不保证全局有序性。

由于队列中index=0的元素为树根，因此出队列时，需要移除。此时需要进行堆调整，来保证堆顶元素最大或者最小。具体逻辑：从根节点的左右子节点，找出最小值，然后与最后一个元素对比，找出min当做树根，左右子树又会找自己左右子树的最小值。在这期间，如果最后一个元素比任何一个子节点小，那么就把最后一个元素放到该位置。

举个例子：  
数组元素：2 4 6 8 10 11
poll一个元素，经过向上堆调整后，最终会变为：4 8 6 11 10

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    // 独占锁
    lock.lock();
    try {
        // 队列为空，返回null
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

size方法：
```java
public int size() {
    final ReentrantLock lock = this.lock;
    // 利用锁保证了size变量的内存可见性(因为size没有被volitile修饰)
    lock.lock();
    try {
        return size;
    } finally {
        lock.unlock();
    }
}
```


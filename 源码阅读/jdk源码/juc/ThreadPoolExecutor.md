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

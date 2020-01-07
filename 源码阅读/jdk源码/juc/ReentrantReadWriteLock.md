## ReentrantReadWriteLock介绍
适用于读多写少、读写分离场景，允许多个线程同时获得锁。读读不互斥，读写互斥，写写互斥,读写锁都是可重入的。  
https://juejin.im/post/5b7d659c6fb9a019fc76dfba
## 源码分析
### 状态值分析：
```java
static final int SHARED_SHIFT   = 16;
// 读锁加1的单位
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
// 15个1
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/** 返回获取读锁的次数  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** 返回该线程写锁的可重入次数  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```


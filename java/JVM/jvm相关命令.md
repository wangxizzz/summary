1.**jmap**
- 可以查看内存的对象信息
- jmap -histo pid

2.**jstack命令**
- jstack命令主要用来查看Java线程的调用堆栈的，可以用来分析线程问题（如死锁）
- **jstack pid 即可查看个线程的情况(包含用户线程与JVM线程)**
    - jstack -l pid: 可以打印出锁的附加信息
    - jstack -F pid : 当jstack没反应时，强制打印
- **线程在JVM的状态：**
    - NEW,未启动的。不会出现在Dump中。
    - RUNNABLE,在虚拟机内执行的。运行中状态，可能里面还能看到locked字样，表明它获得了某把锁。
    - BLOCKED,受阻塞并等待监视器锁。被某个锁(synchronizers)給block住了。
    - WATING,无限期等待另一个线程执行特定操作。等待某个condition或monitor发生，一般停留在park(), wait(), join() 等语句里。
    - TIMED_WATING,有时限的等待另一个线程的特定操作。和WAITING的区别是wait() 等语句加上了时间限制 wait(timeout)。
    - TERMINATED,已退出的。
- 死锁的4个必要条件：
    - **死锁的定义**：死锁是指两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。
    - **发生死锁一定会满足这4个条件，只要这4个条件之一不满足，就无法构成死锁**
    - 互斥条件：一个资源每次只能被一个进程使用。
    - 占有且等待：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
    - 不可剥夺条件:进程已获得的资源，在末使用完之前，不能强行剥夺。
    - 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。
- 预防死锁：
    - 资源分配的有序性
    - ===








1.**int、long、float转String类型：**  
- 第一种方法：s = i + "";   会产生两个String对象
- 第二种方法(推荐)：s = String.valueOf(i);  直接使用String类的静态方法，只产生一个对象

2.**再探内存可见性和原子性：Synchronized和Volatile的比较**
- Synchronized：保证可见性和原子性。Synchronized能够实现原子性(i++操作)和可见性；在Java内存模型中，synchronized规定，线程在加锁时，先清空工作内存→在主内存中拷贝最新变量的副本到工作内存→执行完代码→将更改后的共享变量的值刷新到主内存中→释放互斥锁。
- volatile：保证可见性，但不保证操作的原子性。volatile实现内存可见性是通过store和load指令完成的；也就是对volatile变量执行写操作时，会在写操作后加入一条store指令，即强迫线程将最新的值刷新到主内存中；而在读操作时，会加入一条load指令，即强迫从主内存中读入变量的值。但volatile不保证volatile变量的原子性.
- volatile本质是在告诉JVM当前变量在寄存器中的值是不确定的，使用前，需要先从主存中读取，因此可以实现可见性。而对n=n+1,n++等操作时，volatile关键字将失效，不能起到像synchronized一样的线程同步（原子性）的效果
- volatile不需要加锁，比Synchronized更轻量级，并不会阻塞线程（volatile不会造成线程的阻塞；synchronized可能会造成线程的阻塞。）
- volatile标记的变量不会被编译器优化,而synchronized标记的变量可以被编译器优化（如编译器重排序的优化）.
- volatile是变量修饰符，仅能用于变量，而synchronized是一个方法或块的修饰符。
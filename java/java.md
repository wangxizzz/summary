１、正则表达式写在方法的外面</br>

２、不要使用String.split(),这个方法有bug.推荐使用guava里面的Splitor.</br>

3、推荐使用Logger，不要使用sys.out</br>

４、try resource 
```java
try(Scanner s = new Scanner()) {
    // 随着代码块的结束，会自动关闭s
}  
```
５、static变量与非static变量分开写。  

６、与上下文无关的变量尽量放到方法外面，设置为private static final.


７、自定义bean，如果作为map的key的话，需要重写bean的hashCode()与equals()


８、匿名内部类如果使用lambda表达式写，这个匿名内部类需要时FunctionalInterface.
    匿名内部类会被编译成单独的类，而lambda会被编译成类的私有方法。lambda应用的是invokeDynamic指令。


9、lambda中this代表的是它所在的类，并不是指lambda，而匿名内部类中this表示的是匿名类(innner class).


## 11月29号
10、sleep()方法应用场景：放出cpu资源时使用，不要使用在线程之间的通信。


11、// 共享的结果记录区域　　MutableInt result = new MutableInt(0);√


12、joda时间操作组件？线程安全性问题？
## 11.30


1、线程中断：
```java

public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();
        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                // 调用interrupt()会把中断位reset,也就是false.本来被中断了，中断位应该为true,后来又被重置了.
                // 所以避免使用此方法。
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        // 把标志位改为false(重置)
        interrupt0();
    }
```
**那么怎么实现线程中断？**

首先是线程中之间的协调来是否中断。可以调用```isInterrupted()```来获取中断位flag(如果中断，中断位会标记为true)，然后不断去轮询这个flag,如果位true，就表示中断了，然后就可以执行其他逻辑了。

２、线程池分开使用，比如耗时的任务与简单任务需要分开使用不同的线程池。否则的话，简单任务长期会得不到执行。

３、fork/join框架的应用场景：听讲师说应用到从hdfs上读数据到mysql里，把任务进行拆分时会用到。其实不是有现成的工具sqoop用。。

## 12.1

1、idea中导入eclipse项目报错：

idea上可能出现 Java--Error:java: 无效的标记: -version.  解决办法之一： Settings -> Build -> Java Compiler -> Use compiler 改成eclipse

## 12.03
１、在高并发场景下：
使用悲观锁synchronized.因为乐观锁(AtomicInteger)的很容易失败，然后陷于失败重试，那还不如使用悲观锁让他排队成功。在低并发场景下，CAS成功率要高些。


```java
    public <U> CompletableFuture<U> thenApply(
        Function<? super T,? extends U> fn) {
        return uniApplyStage(null, fn);
    }
    // 会切换线程池，应用场景：这个任务很重，更换线程池
    public <U> CompletableFuture<U> thenApplyAsync(
        Function<? super T,? extends U> fn) {
        return uniApplyStage(asyncPool, fn);
    }
```  

２、慎用ForkJoinPool,需要调用supplyAsync传入自己的线程池。  

３、ConcurrentHashMap在java8中废除了Lock，大量的引入了synchronized,ＣＡＳ,而且对Node加锁，进一步降低了加锁的粒度，提高了并发量。还有就是利用尾插入法，而并非java7的头插入法，解决了HashMap1.7在resize的死循环问题。还有java7的hash函数发生了9次扰动，而１．８只发生２次扰动得到hashCode，虽然这样相对１．７,hashCode可能散列不均匀，但是有了红黑树的改进，时间复杂度依然可观(logn)。  

============================================  

1、keep-alive的作用：
　　减少TCP连接开销。
    TCP慢启动（影响较小）  
2、curl -i https://source.qunarzz.com/common/hf/logo.png
　　查看请求信息。  
3、cache-control: max-age=3110400　采用这个看过期信息。expire会应该机器时间不一致出问题。 
４、大海捞针　建索引的作用。  
5、一条sql只能使用一个索引。(一个单列或者一个复合索引)  
6、日志级别:  
error :一般用于try catch 异常里，输出异常内容。

warn: 多用于一些参数或者返回值信息为空，非正常逻辑的日志输出

info：关键环节的参数或者返回值的打印，常用来定位问题

debug：不重要的日志， 一般不用，用的时候要加log.isDebugEnabled()判断一下，当前debug是否是开启状态
当日志级别在DEBUG以下时，log.debug("hello, this is " + name)就不会执行，从而没有字符串拼接的开销。

JIT在运行时会优化if语句，如果isDebugEnabled()返回false, 则JIT会将整个if块全部去掉。
**1.Java的内部类：**    
```匿名内部类的this代表匿名内部类对象的实例，在lambda中不能引用this对象```

**内部类和外围类的真实关系**   
内部类是个编译时的概念，一旦编译成功后，它就与外围类属于两个完全不同的类（当然它们之间还是有联系的）。对于一个名为Outer的外围类和一个名为Inner的内部类，在编译成功后，会出现这样两个class文件：Outer.class 和 Outer$Inner.class。  

**外部类隐式引用的主要作用**  
静态内部类与非静态内部类之间存在一个最大的区别，就是非静态内部类在编译完成之后会隐含地保存着一个引用，该引用是指向创建它的外围内，但是静态内部类却没有。
这个隐式引用可以由内部类中对外部类成员的调用方式观察到

```java
public class Outer {
    
    private void outerDo() {}
    
    class Inter {
        
        private void innerDo() {
            // 内部类可以直接访问外部类成员，原因在于隐式持有了一个外部类引用
            outerDo();
            // Outer.this 就是内部类隐式持有的外部类引用
            Outer.this.outerDo();
        }
    }
}

// 非静态内部类的内部对象的创建方式：
Outer o = new Outer();
Inter i = o.new Inter();
```
**牢记两个差别：访问权限问题**

一、如是否可以创建静态的成员方法与成员变量(静态内部类可以创建静态的成员，而非静态的内部类不可以).   
二、对于访问外部类的成员的限制(```静态内部类只可以访问外部类中的静态成员变量与成员方法，而非静态的内部类即可以访问所有的外部类成员方法与成员变量```)。

**内部类的应用场景：**
内部类能够帮助Java实现回调，进一步说，它适用于事件驱动的架构。（匿名内部类，FunctionalInterface）  
https://blog.csdn.net/historyasamirror/article/details/6049073  
可以参考lambda替换匿名内部类。

**再简单的说一说被static修饰了的内部类：**  
之所以放在最后，而且我也不愿意详细说，是因为被static修饰的内部类如同被阉割的动物，已经失去了它的活力。因为static内部类是没有包含外部类的this指针的，那么它也就不能够访问外部类的成员。所以，内部类带来的巨大好处和内部类的适用场景它都不具备。我更倾向于将static内部类单独的做为一种情况考虑，而不要将它和普通的内部类混为一谈。

**2.Mysql的子查询与关联查询的区别：**  
子查询：把内层查询结果当作外层查询的比较条件。会有临时表的产生。

示例：
select goods_id,goods_name from goods where goods_id = (select max(goods_id) from goods);

执行子查询时，MYSQL需要创建临时表，查询完毕后再删除这些临时表，所以，子查询的速度会受到一定的影响，这里多了一个创建和销毁临时表的过程。

优化方式：
可以使用连接查询（JOIN）代替子查询，连接查询不需要建立临时表，因此其速度比子查询快。

3.Guava中不可变的Map:

ImmutableMap
```java

// 实现方式 也是在调用改变Map值的方法里直接抛出异常。
public final V put(K k, V v) {
    throw new UnsupportedOperationException();
}
public final V putIfAbsent(K key, V value) {
    throw new UnsupportedOperationException();
}

public final boolean replace(K key, V oldValue, V newValue) {
    throw new UnsupportedOperationException();
}
```

4.Bloom Filter：  
详解布隆过滤器的原理，使用场景和注意事项(误判详解)：https://zhuanlan.zhihu.com/p/43263751

布隆过滤器(Bloom Filter)的原理和实现 ： https://www.cnblogs.com/cpselvis/p/6265825.html

5.如何在几千万的ArryList<String>查找一个手机号，开头是123，末尾是456?
(1) 直接多线程(线程池)拆分List，然后单个线程扫描字符串匹配；  
(2) 如果数据是已经排序的，那么可以直接二分查找。

6.```CompletableFuture```详解：

https://juejin.im/post/5adbf8226fb9a07aac240a67 里面全面的讲述与Futrue, CompletableFuture的应用场景与API,CompletableFuture的异常处理。

7.**子类与父类的构造函数关系：**  
父类：
```java
public class RequestModel {
    /**
     * 表示具体的业务类型
     */
    private String type;
    // 父类重写了构造函数(没有默认无参构造)
    public RequestModel(String type){
        this.type = type;
    }
}
```
子类
```java
public class PreFeeRequestModel extends RequestModel{
   
    public final static String FEE_TYPE = "preFee";

//    public PreFeeRequestModel() {
//        super(FEE_TYPE);
//    }

 // 如果注释掉子类的有参构造函数，就会报错：There is no default constructor available in RequestModel
}
```
```上述错误原因解释：```
Java中只要调用子类的构造函数就一定会调用父类的构造函数，这是毋庸置疑的！有时我们并没有在父类中写有参和无参的构造方法，但是这样我们在定义子类对象时调用子类构造函数时，其实也调用父类的构造函数，这是系统自动为我们添加的“public Pen(){}”。但是如果我们在父类中已经自己定义了有参的构造方法，却没有定义无参的构造方法，那么此时系统是不会为我们自动添加无参的构造方法的，此时程序结果就会提醒你父类没有无参的构造方法，程序就会报错。

8.Spring中的BeanUtils：  
BeanUtils.copyProperties 是一个浅拷贝方法，复制属性时，我们只需要把 DTO 对象和要转化的对象两个的属性值设置为一样的名称，并且保证一样的类型就可以了。  
```如果两个类的属性不一样(属性个数，属性类型)，那么copy后的结果就是属性值为null.```

**9.Hibernate validation:**

更多详细介绍Hibernate validation参考：
https://www.cnblogs.com/mr-yang-localhost/p/7812038.html  全面的介绍Hibernate validation(里面的对@RequestPara参数的校验是正确的，需要创建一个bean, 并且不能使用bindingResult),具体的实现可参照工程design-pattern的refactoring工程的demo01.
 
https://blog.csdn.net/Gred01/article/details/78533390 介绍常用的用法

注意：
- @NotEmpty 用在集合类上面
- @NotBlank 用在String上面
- @NotNull 用在基本类型上
- @Valid被注释的元素是一个对象，需要检查此对象的所有字段值
- @Min(value)被注释的元素必须是一个数字，其值必须大于等于指定的最小值
- @Max(value)被注释的元素必须是一个数字，其值必须小于等于指定的最大值
- @Size(max, min)被注释的元素的大小必须在指定的范围内
- @Future被注释的元素必须是一个将来的日期
- @Pattern(value)被注释的元素必须符合指定的正则表达式
- @Email被注释的元素必须是电子邮箱地址
- @Length(min=, max=)被注释的字符串的大小必须在指定的范围内
- @Range(min=, max=)被注释的元素必须在合适的范围内
如果上述类型与注解使用错误，会抛异常。
https://www.jianshu.com/p/e70cb8ba8f48 列出了各个注解的用法

10.Preconditions参数校验机制：让前置条件判断变得更优雅    
https://blog.csdn.net/zivensonice/article/details/51912188

11.joda time 使用参考：
- code refactoring工程的demo02
- https://blog.csdn.net/qq_36838191/article/details/81296987 介绍了DateTime(相当于Calendar)的操作
- joda time 对日期的格式化    https://www.jianshu.com/p/cb4521194707

### 12.约定优于配置  
Springboot.
```java
// 在Guava中
List<String> list = Lists.newArrayList();
```

13.**Guava中的集合框架API使用:**  

https://www.cnblogs.com/snidget/archive/2013/02/05/2893344.html

14.**transient关键字：**  
https://www.cnblogs.com/lanxuezaipiao/p/3369962.html#top  
也可以参考j2se工程的java高级知识

15.抽象类和Interface相同，不能创建对象(直接new 会变成匿名内部类的方式创建),但是Abstract class有构造函数。

16.**java annotation总结：**  
详细参考网址：  
 介绍了注解自定义方式以及如何访问注解信息：  
https://blog.csdn.net/u013045971/article/details/53433874

**接口提供了以下四个方法来访问Annotation的信息：**

        方法1：<T extends Annotation> T getAnnotation(Class<T> annotationClass): 返回改程序元素上存在的、指定类型的注解，如果该类型注解不存在，则返回null。

        方法2：Annotation[] getAnnotations():返回该程序元素上存在的所有注解。

        方法3：boolean is AnnotationPresent(Class<?extends Annotation> annotationClass):判断该程序元素上是否包含指定类型的注解，存在则返回true，否则返回false.

        方法4：Annotation[] getDeclaredAnnotations()：返回直接存在于此元素上的所有注释。与此接口中的其他方法不同，该方法将忽略继承的注释。（如果没有注释直接存在于此元素上，则返回长度为零的一个数组。）该方法的调用者可以随意修改返回的数组；这不会对其他调用者返回的数组产生任何影响。

扩展：（还是比较简单，固定写法）
- 基于annotation与AOP的日志打印：https://blog.csdn.net/itguangit/article/details/78757782
- 基于annotation的系统登录(IP)拦截：https://blog.csdn.net/aichuanwendang/article/details/54380998

**注解处理器：**

17.**sleep()与wait()、notify()的区别**    
https://blog.csdn.net/u012050154/article/details/50903326  很好的解释了wait()释放同步资源锁，sleep()不释放同步资源锁，notify()不释放锁与唤醒机制。  

1、每个对象都有一个锁来控制同步访问，Synchronized关键字可以和对象的锁交互，来实现同步方法或同步块。sleep()方法正在执行的线程主动让出CPU（然后CPU就可以去执行其他任务），在sleep指定时间后CPU再回到该线程继续往下执行(```注意：sleep方法只让出了CPU，而并不会释放同步资源锁！！！```)；wait()方法则是指当前线程让自己暂时退让出同步资源锁，以便其他正在等待该资源的线程得到该资源进而运行，只有调用了notify()方法，之前调用wait()的线程才会解除wait状态，可以去参与竞争同步资源锁，进而得到执行。（注意：notify的作用相当于叫醒睡着的人，而并不会给他分配任务，就是说notify只是让之前调用wait的线程有权利重新参与线程的调度）；

2、sleep()方法可以在任何地方使用；wait()方法则只能在同步方法或同步块中使用(synchronized修饰的方法或代码块)；

3、sleep()是线程线程类（Thread）的方法，调用会暂停此线程指定的时间，但监控依然保持，不会释放对象锁，到时间自动恢复；wait()是Object的方法，调用会放弃对象锁，进入等待队列，待调用notify()/notifyAll()唤醒指定的线程或者所有线程，才会进入锁池，不再次获得对象锁才会进入运行状态；


```释放锁有两种方式```：  
(1)程序自然离开监视器的范围，即离开synchronized关键字管辖的代码范围  
(2)在synchronized关键字管辖的代码内部调用监视器对象的wait()方法。这里使用wait方法

notify方法并不释放锁

**18、线程间的协作(wait/notify/sleep/yield/join)**  
https://www.cnblogs.com/paddix/p/5381958.html  
线程的5个状态：  
　Java中线程中状态可分为五种：New（新建状态），Runnable（就绪状态），Running（运行状态），Blocked（阻塞状态），Dead（死亡状态）。

　　New：新建状态，当线程创建完成时为新建状态，即new Thread(...)，还没有调用start方法时，线程处于新建状态。

　　Runnable：就绪状态，当调用线程的的start方法后，线程进入就绪状态，等待CPU资源。处于就绪状态的线程由Java运行时系统的线程调度程序(thread scheduler)来调度。

　　Running：运行状态，就绪状态的线程获取到CPU执行权以后进入运行状态，开始执行run方法。

　　Blocked：阻塞状态，线程没有执行完，由于某种原因（如，I/O操作等）让出CPU执行权，自身进入阻塞状态。

　　Dead：死亡状态，线程执行完成或者执行过程中出现异常，线程就会进入死亡状态。

```yield方法```的作用是暂停当前线程，以便其他线程有机会执行，不过不能指定暂停的时间，并且也不能保证当前线程马上停止。yield方法只是将Running状态转变为Runnable状态。

```join方法``` join方法的作用是父线程等待子线程执行完成后再执行，换句话说就是将异步执行的线程合并为同步的线程。

19.
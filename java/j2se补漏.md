**1.Java的内部类：**    

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

8.`
# 总结：
1. singleton作用域：当把一个Bean定义设置为singleton作用域是，Spring IoC容器中只会存在一个共享的Bean实例，并且所有对Bean的请求，只要id与该Bean定义相匹配，则只会返回该Bean的同一实例。值得强调的是singleton作用域是Spring中的缺省作用域。
2. prototype作用域：prototype作用域的Bean会导致在每次对该Bean请求（以程序的方式调用容器的getBean()方法）时会创建一个新的Bean实例。在单例中注入多例bean，最终仍然还是注入的单例。```根据经验，对有状态的Bean应使用prototype作用域，而对无状态的Bean则应该使用singleton作用域。```
3. 对于具有prototype作用域的Bean，有一点很重要，即Spring不能对该Bean的整个生命周期负责。具有prototype作用域的Bean创建后交由调用者负责销毁对象回收资源。
4. 简单的说：
singleton 只有一个实例，也即是单例模式。
prototype访问一次创建一个实例，相当于new。 
5. 应用场合：  
1.需要回收重要资源(数据库连接等)的事宜配置为singleton，如果配置为prototype需要应用确保资源正常回收。  
2.有状态的Bean配置成singleton会引发未知问题，可以考虑配置为prototype。

# 实际问题
项目中，报表导出涉及到了在同一个类的两个不同方法中，都有相同的查询数据库的操作，一个方法是用于获取内容，一个是用于获取条数的，大概类似于这样：
```java
@Service
public class MyReportExporter extends AbstractReportExporter{
    @Override
    protected DataResp getData(Param param) {
        List records = myService.queryList(param);//查询db
        return wrapResp(records);
    }

    @Override
    protected int getCount(Param param) {
        return myService.queryList(param).size();//查询db
    }
}
```
由于是继承的父类统一处理，因此没办法单独优化这个步骤。在父类的统一处理过程中，会多次调用getCount方法，这样每处理一次，就需要多次查询数据库。

这是会想到，可以用私有全局变量将查询结果存起来。

# 使用原型
在Spring中，@Service默认都是单例的。用了私有全局变量，若不想影响下次请求，就需要用到原型模式，即@Scope(“prototype”)

所谓单例，就是Spring的IOC机制只创建该类的一个实例，每次请求，都会用这同一个实例进行处理，因此若存在全局变量，本次请求的值肯定会影响下一次请求时该变量的值。
原型模式，指的是每次调用时，会重新创建该类的一个实例，比较类似于我们自己自己new的对象实例。

通过查看@Scope我们可以看到，默认的模式：singleton
```java
public @interface Scope {
String value() default ConfigurableBeanFactory.SCOPE_SINGLETON;
...
}
```

通过如下方式，可以将该类设置为原型模式
```java
@Service
@Scope("prototype")
public class MyReportExporterextends AbstractReportExporter{
    ...
}
```
# prototype陷阱
在进行以上改动后，运行发现并没有生效，依然是一个实例。这说明只加一个@Scope注解还不够。

在调用改service的controller层，是这样注入的：
```java
    @Autowired
    private MyReportExporter myReportExporter;
```
而controller同样是默认单例的，因此只实例化了一个controller对象，在其中依赖注入的MyReportExporter对象也就只会实例化一次。

在不想改变controller单例模式的情况下，可以如下修改：
放弃使用@Autowired方式，改用getBean方式：
```java
private static ApplicationContext applicationContext;
MyReportExporter myReportExporter = applicationContext.getBean(MyReportExporter.class);
```

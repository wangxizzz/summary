1.**Spring bean加载中的Map变量：**
- singletonObjects
    - new ConcurrentHashMap<>()，用于保存beanName与bean实例的关系
- earlySingletonObjects
    - new HashMap<>，用于保存beanName与bean实例的关系，与singletonObjects不同的是，当一个单例bean被放入earlySingletonObjects中后，那么当bean还在加载中，就可以通过getBean()取到了，其目的是用来就行循环检测。
- singletonFactories
    - new HashMap()，用于保存beanName与bean的工厂关系。
- registeredSingletons：
    - new LinkedHashSet<String>()，用于顺序保存已经注册的beanName

2.**</bean>标签的属性介绍：**
- https://blog.csdn.net/qq_41971087/article/details/82225456
- https://blog.csdn.net/xiao1_1bing/article/details/81084111


**spring如何解决bean的循环依赖问题？**
- 通过xml中构造器循环依赖是无法解决的, 只能抛出BeanCurrentlyInCreationException。
- 通过xml中配置的setter注入方式。对于Setter注入造成的依赖是通过 Spring 容器 提前暴露刚完成构造器注入但未完成其他步骤(如 setter注入)的 bean来完成的，而且只能解决```单例```作用域的 bean 循环依赖，对于“ prototype”作用域 bean, Spring 容器无法完成依赖 注入，因为 Spring 容器不进行缓 存“prototype”作用域的 bean，因此无法提前暴露一个创建中的 bean。

**Spring的总体架构**：  
- 

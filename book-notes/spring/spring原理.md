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

**Spring的总体架构**：  
- 

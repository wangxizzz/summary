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
- 通过xml中配置的setter注入方式，spring的解决办法：
    - 对于Setter注入造成的依赖是通过 Spring 容器 提前暴露刚完成构造器注入但未完成其他步骤(如 setter注入)的 bean来完成的，而且只能解决```单例```作用域的 bean 循环依赖，对于“ prototype”作用域 bean, Spring 容器无法完成依赖 注入，因为 Spring 容器不进行缓 存“prototype”作用域的 bean，因此无法提前暴露一个创建中的 bean。
    - A依赖于B，B依赖于A，基本可以理清 Spring 处理循环依赖的解决办法， 在 B 中创 建依赖 A 时通过 ObjectFactory 提供的实例化方法来中断 A 中的属性填充 ， 使 B 中持有的 A 仅仅是刚刚初始化并没有填充任何属性的 A， 而这正初始化 A 的步骤还是在最开始创建 A 的 时候进行的，但是因为 A 与 B 中的 A 所表示的属性地址是一样的 ， 所以在 A 中创建好的属性 填充自然可 以通过 B 中的 A 获取，这样就解决了循环依赖 的问题。

**Spring的总体架构**：  
- 

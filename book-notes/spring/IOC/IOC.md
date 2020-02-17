# IOC分析
## 容器初始化阶段--BeanDefinition的注册
> 入口在org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.Resource)  

解析步骤如下：
1. 把配置文件封装成统一的抽象资源文件```Resource```,利用Resource获取文件输入流，再利用ResourceLoader加载配置文件，获取xml Document对象。
2. 解析xml元素，把元素属性封装成BeanDefinition，然后注册到容器beanDefinitionMap中(key为beanName，value是BeanDefinition)。

## Bean的加载阶段
加载步骤如下：
1. 经过容器初始化阶段后，应用程序中定义的 bean 信息已经全部加载到系统中了，当我们显示或者隐式地调用 ```BeanFactory#getBean(...) ```方法时，则会触发加载 Bean 阶段。
2. 在这阶段，容器会首先检查所请求的对象是否已经初始化完成了，如果没有，则会根据注册的 Bean 信息实例化请求的对象，并为其注册依赖，然后将其返回给请求方。

## Bean加载核心方法：doGetBean 分析

> 由于方法过长，因此采用逐段分析。源码位于 ```AbstractBeanFactory.java```
### 从单例缓存中加载Bean
```java
    // 从缓存中或者实例工厂中获取Bean对象
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isTraceEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        // 完成 FactoryBean 的相关处理，并用来获取 FactoryBean 的处理结果
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
```
- 我们知道单例模式的 Bean 在整个过程中只会被创建一次。第一次创建后会将该 Bean 加载到缓存中。后面，在获取 Bean 就会直接从单例缓存中获取。
- 如果从缓存中得到了 Bean 对象，则需要调用 ```getObjectForBeanInstance(Object beanInstance, String name, String beanName, RootBeanDefinition mbd) ```方法，对 Bean 进行实例化处理。因为，缓存中记录的是最原始的 Bean 状态，我们得到的不一定是我们最终想要的 Bean 。另外，```FactoryBean 的用途如下```：一般情况下，Spring 通过反射机制利用 bean 的 class 属性指定实现类来实例化 bean 。某些情况下，实例化 bean 过程比较复杂，如果按照传统的方式，则需要在 中提供大量的配置信息，配置方式的灵活性是受限的，这时采用编码的方式可能会得到一个简单的方案。Spring 为此提供了一个 FactoryBean 的工厂类接口，用户可以通过实现该接口定制实例化 bean 的逻辑。FactoryBean 接口对于 Spring 框架来说战友重要的地址，Spring 自身就提供了 70 多个 FactoryBean 的实现。它们隐藏了实例化一些复杂 bean 的细节，给上层应用带来了便利。

<a href="getSingleton.md">详情参照从缓存中加载单例bean</a>

### 原型模式依赖检查
```java
// 因为 Spring 只解决单例模式下得循环依赖，在原型模式下如果存在循环依赖则会抛出异常。
    if (isPrototypeCurrentlyInCreation(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
    }
```
Spring 只处理单例模式下得循环依赖，对于原型模式的循环依赖直接抛出异常。主要原因还是在于，和 Spring 解决循环依赖的策略有关。

- 对于单例( Singleton )模式， Spring 在创建 Bean 的时候并不是等 Bean 完全创建完成后才会将 Bean 添加至缓存中，而是 不等 Bean 创建完成就会将创建 Bean 的 ObjectFactory 提早加入到缓存中，这样一旦下一个 Bean 创建的时候需要依赖 bean 时则直接使用 ObjectFactroy 。
- 但是原型( Prototype )模式，我们知道是没法使用缓存的，所以 Spring 对原型模式的循环依赖处理策略则是不处理。

### 从 parentBeanFactory 获取 Bean
> 背景：如果从单例缓存中没有对应的bean,那么就需要通过BeanDefinition加载Bean实例。
```java
// 如果当前容器中没有找到，则从父类容器中加载
BeanFactory parentBeanFactory = getParentBeanFactory();
// 首先判断是否存在父工厂 并且 判断当前容器是否存在beanName对应的BeanDefinition.
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // Not found -> check parent.
    String nameToLookup = originalBeanName(name);
    // 如果，父类容器为 AbstractBeanFactory ，直接递归查找
    if (parentBeanFactory instanceof AbstractBeanFactory) {
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                nameToLookup, requiredType, args, typeCheckOnly);
    // 用明确的 args 从 parentBeanFactory 中，获取 Bean 对象
    } else if (args != null) {
        // Delegation to parent with explicit args.
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    // 用明确的 requiredType 从 parentBeanFactory 中，获取 Bean 对象
    } else if (requiredType != null) {
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    // 直接使用 nameToLookup 从 parentBeanFactory 获取 Bean 对象
    } else {
        return (T) parentBeanFactory.getBean(nameToLookup);
    }
}
```

### 指定的 Bean 标记为已经创建或即将创建
```java
// 如果不是仅仅做类型检查则是创建 bean，这里要进行记录
    if (!typeCheckOnly) {
        markBeanAsCreated(beanName);
    }
```

### 获取 BeanDefinition
```java
// 从容器中获取 beanName 相应的 GenericBeanDefinition 对象，并将其转换为 RootBeanDefinition 对象
final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
// 检查给定的合并的 BeanDefinition
checkMergedBeanDefinition(mbd, beanName, args);
```
>> 因为从 XML 配置文件中读取到的 Bean 信息是存储在GenericBeanDefinition 中的。但是，所有的 Bean 后续处理都是针对于 RootBeanDefinition 的，所以这里需要进行一个转换。  
>> 转换的同时，如果父类 bean 不为空的话，则会一并合并父类的属性。

### 依赖 Bean 处理
```java
// 处理所依赖的bean
    String[] dependsOn = mbd.getDependsOn();
    // 如果存在依赖则需要递归实例化依赖的 bean
    if (dependsOn != null) {
        for (String dep : dependsOn) {
            // 若给定的依赖 bean 已经注册为依赖给定的 bean  即为循环依赖的情况
            if (isDependent(beanName, dep)) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
            }
            // 缓存中注册依赖的 bean，所以在实例化本 bean 前，需要将它所依赖的 bean 先初始化
            registerDependentBean(dep, beanName);
            try {
                // 加载依赖bean
                getBean(dep);
            }
            catch (NoSuchBeanDefinitionException ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
            }
        }
    }
```
- 每个 Bean 都不是单独工作的，它会依赖其他 Bean，其他 Bean 也会依赖它。  
- 对于依赖的 Bean ，它会优先加载，所以，在 Spring 的加载顺序中，在初始化某一个 Bean 的时候，首先会初始化这个 Bean 的依赖

### 不同作用域的 Bean 实例化
```真正创建Bean的地方，详情请戳 ```<a href="createBean.md">bean的创建</a>
```java
// 创建bean实例，如果是单例模式
if (mbd.isSingleton()) {
    // 第二个参数的回调接口，接口是 org.springframework.beans.factory.ObjectFactory#getObject
    // 接口实现的方法是 createBean(beanName, mbd, args)
    sharedInstance = getSingleton(beanName, () -> {
        try {
            // 真正加载bean的方法
            return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            destroySingleton(beanName);
            throw ex;
        }
    });
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
// 原型模式
else if (mbd.isPrototype()) {
    // It's a prototype -> create a new instance.
    Object prototypeInstance = null;
    try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
    }
    finally {
        afterPrototypeCreation(beanName);
    }
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}

else {
    // 以上两种方式都不是，在指定 scope 上创建实例
    String scopeName = mbd.getScope();
    final Scope scope = this.scopes.get(scopeName);
    if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
    }
    try {
        Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            try {
                return createBean(beanName, mbd, args);
            }
            finally {
                afterPrototypeCreation(beanName);
            }
        });
        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    }
    catch (IllegalStateException ex) {
        throw new BeanCreationException(beanName,
                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                ex);
    }
}
```
- Spring Bean默认的作用域是singleton.当然，还有其他作用域，如 prototype、request、session 等
- 不同的作用域会有不同的初始化策略。

### Bean类型转换
```java
// 检查需要的类型是否符合 bean 的实际类型
if (requiredType != null && !requiredType.isInstance(bean)) {
    // 如果不符合，那么做一层转换
    try {
        T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
        if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
        return convertedBean;
    }
    catch (TypeMismatchException ex) {
        if (logger.isTraceEnabled()) {
            logger.trace("Failed to convert bean '" + name + "' to required type '" +
                    ClassUtils.getQualifiedName(requiredType) + "'", ex);
        }
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
    }
}
```
- 在调用 #doGetBean(...) 方法时，有一个 requiredType 参数。该参数的功能就是将返回的 Bean 转换为 requiredType 类型。
- 当然就一般而言，我们是不需要进行类型转换的，也就是 requiredType 为空（比如 #getBean(String name) 方法）。但有，可能会存在这种情况，比如我们返回的 Bean 类型为 String ，我们在使用的时候需要将其转换为 Integer，那么这个时候 requiredType 就有用武之地了。**```当然我们一般是不需要这样做的。```**
# createBean，bean的创建

## createBean抽象方法：
```java
protected abstract Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException;
```
- 该方法定义在 AbstractBeanFactory 中，其含义是根据给定的 BeanDefinition 和 args 实例化一个 Bean 对象。
- 如果该 BeanDefinition 存在父类，则该 BeanDefinition 已经合并了父类的属性。
所有 Bean 实例的创建，都会委托给该方法实现。
- 该方法接受三个方法参数：
    - beanName ：bean 的名字。
    - mbd ：已经合并了父类属性的（如果有的话）BeanDefinition 对象。
    - args ：用于构造函数或者工厂方法创建 Bean 实例对象的参数。

## createBean默认实现

```java

```
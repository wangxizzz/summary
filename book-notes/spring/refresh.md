# org.springframework.context.support.AbstractApplicationContext#refresh
```这个方法是spring初始化的入口：```
```java
// 此时 ClassPathXmlApplicationContext 的构造函数
public ClassPathXmlApplicationContext(
        String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
        throws BeansException {

    super(parent);
    // 注释 1.1 获取资源文件
    setConfigLocations(configLocations);
    if (refresh) {
        // 调用
        refresh();
    }
}

// 其他ApplicationContext构造仍会调用 refresh方法
```

# refresh源码分析
```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 准备刷新上下文环境
        prepareRefresh();

        // 创建并初始化 BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context. 对beanFactory做一些特殊处理
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 允许在上下文子类中对 bean工厂 进行后处理，要求在子类中实现该方法
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            // 注释 6.1  注册并执行 beanFactoryPostProcessor
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            // 注释 7.1 注册 beanPostProcessor 主体是 bean，这一步中没有执行，只是注册动作
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            // 初始化消息资源
            initMessageSource();

            // Initialize event multicaster for this context.
            // 初始化事件传播器
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // 让子类实现
            onRefresh();

            // Check for listener beans and register them.
            // 检查并注册监听器(比如注册ApplicationContextEvent的事件监听器)
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // 注释 4.2 实例化单例
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            // 最后一步，发送广播事件，保证对应的监听器可以做进一步的逻辑处理(此时bean全部实例化完成)
            finishRefresh();
        }
        ....
    }
}
```

## obtainFreshBeanFactory
```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 刷新 BeanFactory,方法的核心任务就是创建 BeanFactory 并对其就行一番初始化
    refreshBeanFactory();
    // 获取BeanFactory
    return getBeanFactory();
}

protected final void refreshBeanFactory() throws BeansException {
    // 在更新时，如果发现已经存在，将会把之前的 bean 清理掉，并且关闭老 bean 容器
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 创建 BeanFactory 对象.本质是new 了一个DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 指定序列化编号
        beanFactory.setSerializationId(getId());
        // 定制 BeanFactory 设置相关属性
        customizeBeanFactory(beanFactory);
        // 开始加载BeanDefinition （bean 注册）最终会初始化 Map<String, BeanDefinition>，bean没有实例化
        loadBeanDefinitions(beanFactory);
        // 由于 beanFactory 是公共变量，存在多线程操作，所以加锁操作，避免混乱修改
        synchronized (this.beanFactoryMonitor) {
            // 设置 Context 的 BeanFactory
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

## initApplicationEventMulticaster
```java
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 如有有自己注册class Name 是 applicationEventMulticaster，使用自定义广播器
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        if (logger.isTraceEnabled()) {
            logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    }
    else {
        // 没有自定义，使用默认的事件广播器
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
        if (logger.isTraceEnabled()) {
            logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
                    "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
        }
    }
}
```
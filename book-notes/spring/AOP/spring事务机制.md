# 主流程介绍
众所周知，Spring事务采用AOP的方式实现，我们从TransactionAspectSupport这个类开始f分析。
1. 获取事务的属性（@Transactional注解中的配置）
2. 加载配置中的TransactionManager.
3. 获取收集事务信息TransactionInfo
4. 执行目标方法
5. 出现异常，尝试处理。
6. 清理事务相关信息
7. 提交事务

基于AOP拦截，使用threadLocal hold住当前连接，业务处理需要从这里拿到连接去处理，最后一起提交事务或回滚就行。

# 源码分析

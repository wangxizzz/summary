1.**提供方与消费方的id问题：**  
服务的提供方
```xml
<dubbo:service interface="com.demo.service.UserService"
    <!-- 这个ref 的值表示，真正提供服务接口实现的bean的名字(这个bean的名字可以在@Component(value="")来指定，也可以通过xml配置bean -->
    ref="userServiceImpl01" timeout="1000" version="1.0.0">
    <dubbo:method name="getUserAddressList" timeout="1000"></dubbo:method>
</dubbo:service>
<!-- bean的实现者，采用xml配置 -->
<bean id="userServiceImpl01" class="service.impl.UserServiceImpl"></bean>

<dubbo:service interface="com.demo.service.UserService"
<!-- 同样的服务，接口的实现不同，version为2.0.0 -->
    ref="userServiceImpl02" timeout="1000" version="2.0.0">
    <dubbo:method name="getUserAddressList" timeout="1000"></dubbo:method>
</dubbo:service>
<bean id="userServiceImpl02" class="service.impl.UserServiceImpl2"></bean>
```
服务的消费方：
```xml
<dubbo:reference interface="com.demo.service.UserService"
<!-- consumer端的id表示在consumer端的业务逻辑代码注入此bean所需要的名字(比如Autoware,Resource,Reference)，跟provider端的bean没有关系 -->
    id="userService" timeout="5000" retries="3" version="1.0.0" stub="service.impl.UserServiceStub">
    <!-- <dubbo:method name="getUserAddressList" timeout="1000"></dubbo:method> -->
</dubbo:reference>

<!-- 对 id="userService" 的解释 -->
public class OrderServiceImpl implements OrderService {

	@Autowired
	private UserService userService;  // 在这里需要用到id="userService"
}
```
如果在接口服务升级了，如何引用最新的服务？  
```利用服务的version字段```。当然，新旧服务的实现肯定都是实现相同的接口，在provider端把旧的服务的version改为1.0.0,把新的服务version改为2.0.0，这样在consumer端，把服务的引用配置的version改为想要引用的版本即可。


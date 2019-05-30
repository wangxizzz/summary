1.限流的策略：  
https://mp.weixin.qq.com/s/HXKInG0o42lTGKtZiveFdA  
https://www.infoq.cn/article/microservice-interface-rate-limit  
https://segmentfault.com/a/1190000016240755 Guava的限流策略。

滑动窗口的理解：把一分钟的时间周期平均划为60等分，1s一个窗口，然后有一个count[60]的数组，来记录每个秒的窗口的请求数量(就是每个小窗口都含有自己的计数器)，在每个请求到达时，判断所有小窗口的请求和是否超过系统阈值。在新的时间刻来到，滑动窗口向前移动。
```java
全局数组 链表[]  counterList = new 链表[切分的滑动窗口数量];
//有一个定时器，在每一次统计时间段起点需要变化的时候就将索引0位置的元素移除，并在末端追加一个新元素。
int sum = counterList.Sum();  // 总的请求量(数组各元素相加)
if(sum > 限流阈值) {
    return; //不继续处理请求。
}
int 当前索引 = 当前时间的秒数 % 切分的滑动窗口数量;
counterList[当前索引]++;
// do something...
```
2.TCP滑动窗口：  
https://blog.csdn.net/wdscq1234/article/details/52444277

3.Redis的热点数据问题：  
https://yq.aliyun.com/articles/404817

4.缓存穿透与缓存雪崩问题：  
https://blog.csdn.net/wang0112233/article/details/79558612

5.Redis缓存一致性问题：  

6.epoll与select

7.基于Redis的分布式锁。

8.



## 1.分布式锁介绍：
### 1.1 redis分布式锁实现：
```java
// redisTemplate的加锁方式，利用setIfAbsent模拟锁
ValueOperations<String, String> operations = stringRedisTemplate.opsForValue();
Boolean setResult = operations.setIfAbsent(lockKey, requestId, Duration.ofMillis(expireTime));

// jedis的方式
String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
if (LOCK_SUCCESS.equals(result)) {
    return true;
}
return false;
```

## 2.Redis分布式锁的问题和解决办法：
### 2.1 除了网上介绍的问题：异常导致死锁、A错解B客户端锁、解锁的原子性问题。
### 2.2 **还有一个重大的问题：**
- 问题描述：
    - A客户端获得了锁,并且设置了超时时间为```2s```，业务逻辑是从zset中取出指定大小的数据，然后删除取出的数据，然后释放锁。在并发低情况下，redis执行命令只要```1ms```左右，但是在并发非常高的情况下，即使A客户端加锁成功了，业务逻辑执行完了，然后开始执行释放锁的命令，此时A发送lua命令(原子释放锁)到redis，但是redis单线程，并发又很高，那么A的释放锁的命令，就会长时间得不到执行(此时redis的其他客户端仍然在不断竞争锁，发送setIfAbsent命令,可能还有重试机制)，在系统的监控中发现，最长时间到达了```10s```才执行A的释放锁的命令，此时A的过期时间早就到了，因此此时再释放A客户端的锁，就会造成释放锁失败。
- 上述问题为什么要加分布式锁？
    - 因为从redis中取出size大小的数据与删除size大小的数据，不是原子性的。redis是共享资源。
- 解决办法：
    - 把业务代码封装成lua脚本，原子执行业务逻辑。然后删除redis分布式锁即可解决上述问题(可以根源上解决)。
    - 使用zk实现分布式锁，但是会与Client与Server端的Session过期时间有关系。
    - radisson起一个监听过期时间的线程，如果过期就会延长ttl，但是释放锁的命令是在服务端阻塞住了，貌似无法解决上述问题。

lua脚本如下：
```lua
local json = cjson
local result = redis.call('zrange', KEYS[1], ARGV[1], ARGV[2])
if next(result) ~= nil then
    -- unpack(result)，把result变为list
    redis.call('zrem', KEYS[1], unpack(result))
    return json.encode(result)
else
    return ''
end
```
    
## 关于Redis分布式锁的思考：
- https://juejin.im/post/58b3a93c1b69e60058b49767
- 这篇文章记录了redis lock的所有缺点与问题
- 给出了不同场景解决方案的选取。
- 由国外论文翻译而来
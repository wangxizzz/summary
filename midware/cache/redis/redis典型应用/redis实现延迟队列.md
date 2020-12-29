# redission的延迟队列实现

## 简易版实现
com.code.refactoring.redis相关.redisTemplate操作.RedisTemplateUtilTest#testDelayQueue

## RDelayedQueue.offer
```java
public class RedissonDelayedQueue<V> extends RedissonExpirable implements RDelayedQueue<V> {

    private final QueueTransferService queueTransferService;
    private final String channelName;
    private final String queueName;
    private final String timeoutSetName;
    
    protected RedissonDelayedQueue(QueueTransferService queueTransferService, Codec codec, final CommandAsyncExecutor commandExecutor, String name) {
        super(codec, commandExecutor, name);
        channelName = prefixName("redisson_delay_queue_channel", getName());
        queueName = prefixName("redisson_delay_queue", getName());
        timeoutSetName = prefixName("redisson_delay_queue_timeout", getName());
        
        //QueueTransferTask task = ......
        
        queueTransferService.schedule(queueName, task);
        
        this.queueTransferService = queueTransferService;
    }

    public void offer(V e, long delay, TimeUnit timeUnit) {
        get(offerAsync(e, delay, timeUnit));
    }
    
    public RFuture<Void> offerAsync(V e, long delay, TimeUnit timeUnit) {
        long delayInMs = timeUnit.toMillis(delay);
        long timeout = System.currentTimeMillis() + delayInMs;
     
        long randomId = PlatformDependent.threadLocalRandom().nextLong();
        return commandExecutor.evalWriteAsync(getName(), codec, RedisCommands.EVAL_VOID,
                "local value = struct.pack('dLc0', tonumber(ARGV[2]), string.len(ARGV[3]), ARGV[3]);" 
              + "redis.call('zadd', KEYS[2], ARGV[1], value);"
              + "redis.call('rpush', KEYS[3], value);"
              // if new object added to queue head when publish its startTime 
              // to all scheduler workers 
              + "local v = redis.call('zrange', KEYS[2], 0, 0); "
              + "if v[1] == value then "
                 + "redis.call('publish', KEYS[4], ARGV[1]); "
              + "end;"
                 ,
              Arrays.<Object>asList(getName(), timeoutSetName, queueName, channelName), 
              timeout, randomId, encode(e));
    }

    public ByteBuf encode(Object value) {
        if (commandExecutor.isRedissonReferenceSupportEnabled()) {
            RedissonReference reference = RedissonObjectFactory.toReference(commandExecutor.getConnectionManager().getCfg(), value);
            if (reference != null) {
                value = reference;
            }
        }
        
        try {
            return codec.getValueEncoder().encode(value);
        } catch (IOException e) {
            throw new IllegalArgumentException(e);
        }
    }

    public static String prefixName(String prefix, String name) {
        if (name.contains("{")) {
            return prefix + ":" + name;
        }
        return prefix + ":{" + name + "}";
    }

    //......
}

```

- 这里使用的是一段lua脚本，其中keys参数数组有四个值，KEYS[1]为getName(), KEYS[2]为timeoutSetName, KEYS[3]为queueName, KEYS[4]为channelName
- 变量有三个，ARGV[1]为timeout，ARGV[2]为randomId，ARGV[3]为encode(e)
- 这段lua脚本对timeoutSetName的zset添加一个结构体，其score为timeout值；对queueName的list的表尾添加结构体；然后判断timeoutSetName的zset的第一个元素是否是当前的结构体，如果是则对channel发布timeout消息

## queueTransferService.schedule 
```java
        QueueTransferTask task = new QueueTransferTask(commandExecutor.getConnectionManager()) {
            
            @Override
            protected RFuture<Long> pushTaskAsync() {
                return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_LONG,
                        "local expiredValues = redis.call('zrangebyscore', KEYS[2], 0, ARGV[1], 'limit', 0, ARGV[2]); "
                      + "if #expiredValues > 0 then "
                          + "for i, v in ipairs(expiredValues) do "
                              + "local randomId, value = struct.unpack('dLc0', v);"
                              + "redis.call('rpush', KEYS[1], value);"
                              + "redis.call('lrem', KEYS[3], 1, v);"
                          + "end; "
                          + "redis.call('zrem', KEYS[2], unpack(expiredValues));"
                      + "end; "
                        // get startTime from scheduler queue head task
                      + "local v = redis.call('zrange', KEYS[2], 0, 0, 'WITHSCORES'); "
                      + "if v[1] ~= nil then "
                         + "return v[2]; "
                      + "end "
                      + "return nil;",
                      Arrays.<Object>asList(getName(), timeoutSetName, queueName), 
                      System.currentTimeMillis(), 100);
            }
            
            @Override
            protected RTopic<Long> getTopic() {
                return new RedissonTopic<Long>(LongCodec.INSTANCE, commandExecutor, channelName);
            }
        };
        
        queueTransferService.schedule(queueName, task);

```
- RedissonDelayedQueue构造器里头对QueueTransferTask进行调度
- 调度执行的是pushTaskAsync方法，主要就是将到期的元素从元素队列移到目标队列
- 这里使用一段lua脚本，KEYS[1]为getName()，KEYS[2]为timeoutSetName，KEYS[3]为queueName；ARGV[1]为当前时间戳，ARGV[2]为100
- 这里调用zrangebyscore，对timeoutSetName的zset使用timeout参数进行排序，取得分介于0和当前时间戳的元素，取前200条
- 如果有值表示该元素需要移交到目标队列，然后调用rpush移交到目标队列，再调用lrem从元素队列移除，最后在从timeoutSetName的zset中删除掉已经处理的这些元素
- 处理完过元素转移之后，再取timeoutSetName的zset的第一个元素的得分返回，如果没有返回nil



参考：https://juejin.cn/post/6844903683247833102#heading-0

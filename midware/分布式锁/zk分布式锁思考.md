## 如何使用zookeeper实现分布式锁？
在描述算法流程之前，先看下zookeeper中几个关于节点的有趣的性质：
- 有序节点：假如当前有一个父节点为/lock，我们可以在这个父节点下面创建子节点；zookeeper提供了一个可选的有序特性，例如我们可以创建子节点“/lock/node-”并且指明有序，那么zookeeper在生成子节点时会根据当前的子节点数量自动添加整数序号，也就是说如果是第一个创建的子节点，那么生成的子节点为/lock/node-0000000000，下一个节点则为/lock/node-0000000001，依次类推。
- 临时节点：客户端可以建立一个临时节点，在会话结束或者会话超时后，zookeeper会自动删除该节点。
事件监听：在读取数据时，我们可以同时对节点设置事件监听，当节点数据或结构变化时，zookeeper会通知客户端。当前zookeeper有如下四种事件：1）节点创建；2）节点删除；3）节点数据修改；4）子节点变更。

### 下面描述使用zookeeper实现分布式锁的算法流程，假设锁空间的根节点为/lock：
- 客户端连接zookeeper，并在/lock下创建临时的且有序的子节点，第一个客户端对应的子节点为/lock/lock-0000000000，第二个为/lock/lock-0000000001，以此类推。
- 客户端获取/lock下的子节点列表，判断自己创建的子节点是否为当前子节点列表中序号最小的子节点，如果是则认为获得锁，否则监听/lock的子节点变更消息，获得子节点变更通知后重复此步骤直至获得锁；
- 执行业务代码；
- 完成业务流程后，删除对应的子节点释放锁。

步骤1中创建的临时节点能够保证在故障的情况下锁也能被释放，考虑这么个场景：假如客户端a当前创建的子节点为序号最小的节点，获得锁之后客户端所在机器宕机了，客户端没有主动删除子节点；如果创建的是永久的节点，那么这个锁永远不会释放，导致死锁；由于创建的是临时节点，客户端宕机后，过了一定时间zookeeper没有收到客户端的心跳包判断会话失效，将临时节点删除从而释放锁。

另外细心的朋友可能会想到，在步骤2中获取子节点列表与设置监听这两步操作的原子性问题，考虑这么个场景：客户端a对应子节点为/lock/lock-0000000000，客户端b对应子节点为/lock/lock-0000000001，客户端b获取子节点列表时发现自己不是序号最小的，但是在设置监听器前客户端a完成业务流程删除了子节点/lock/lock-0000000000，客户端b设置的监听器岂不是丢失了这个事件从而导致永远等待了？这个问题不存在的。因为zookeeper提供的API中设置监听器的操作与读操作是原子执行的，也就是说在读子节点列表时同时设置监听器，保证不会丢失事件。  

最后，对于这个算法有个极大的优化点：假如当前有1000个节点在等待锁，如果获得锁的客户端释放锁时，这1000个客户端都会被唤醒，这种情况称为```“羊群效应”```；在这种羊群效应中，zookeeper需要通知1000个客户端，这会阻塞其他的操作，最好的情况应该只唤醒新的最小节点对应的客户端。应该怎么做呢？在设置事件监听时，每个客户端应该对刚好在它之前的子节点设置事件监听，例如子节点列表为/lock/lock-0000000000、/lock/lock-0000000001、/lock/lock-0000000002，序号为1的客户端监听序号为0的子节点删除消息，序号为2的监听序号为1的子节点删除消息。

### 所以调整后的分布式锁算法流程如下：
- 客户端连接zookeeper，并在/lock下创建临时的且有序的子节点，第一个客户端对应的子节点为/lock/lock-0000000000，第二个为/lock/lock-0000000001，以此类推。
- 客户端获取/lock下的子节点列表，判断自己创建的子节点是否为当前子节点列表中序号最小的子节点，如果是则认为获得锁，否则监听刚好在自己之前一位的子节点删除消息，获得子节点变更通知后重复此步骤直至获得锁；
- 执行业务代码；
- 完成业务流程后，删除对应的子节点释放锁。

## Zk分布式锁的缺点：
- Zookeeper实现的分布式锁其实存在一个缺点，那就是性能上可能并没有缓存服务(Redis)那么高。因为每次在创建锁和释放锁的过程中，都要动态创建、销毁瞬时节点来实现锁功能。ZK中创建和删除节点只能通过Leader服务器来执行，然后将数据同不到所有的Follower机器上。

## Curator的源码分析
zk分布式锁实现：
```java
public static void main(String[] args) throws Exception {
    //创建zookeeper的客户端
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.newClient("10.21.41.181:2181,10.21.42.47:2181,10.21.49.252:2181", retryPolicy);
    client.start();

    //创建分布式锁, 锁空间的根节点路径为/curator/lock
    InterProcessMutex mutex = new InterProcessMutex(client, "/curator/lock");
    mutex.acquire();
    //获得了锁, 进行业务流程
    System.out.println("Enter mutex");
    //完成业务流程, 释放锁
    mutex.release();
    //关闭客户端
    client.close();
}
```

下面来分析下获取锁的源码实现。acquire的方法如下：
```java
/*
 * 获取锁，当锁被占用时会阻塞等待，这个操作支持同线程的可重入（也就是重复获取锁），acquire的次数需要与release的次数相同。
 * @throws Exception ZK errors, connection interruptions
 */
@Override
public void acquire() throws Exception
{
    if ( !internalLock(-1, null) )
    {
        throw new IOException("Lost connection while trying to acquire lock: " + basePath);
    }
}
```

```java
private boolean internalLock(long time, TimeUnit unit) throws Exception
{

    //这里处理同线程的可重入性，如果已经获得锁，那么只是在对应的数据结构中增加acquire的次数统计，直接返回成功
    Thread currentThread = Thread.currentThread();
    LockData lockData = threadData.get(currentThread);
    if ( lockData != null )
    {
        // re-entering
        lockData.lockCount.incrementAndGet();
        return true;
    }

    //这里才真正去zookeeper中获取锁
    String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
    if ( lockPath != null )
    {
        //获得锁之后，记录当前的线程获得锁的信息，在重入时只需在LockData中增加次数统计即可
        LockData newLockData = new LockData(currentThread, lockPath);
        threadData.put(currentThread, newLockData);
        return true;
    }

    //在阻塞返回时仍然获取不到锁，这里上下文的处理隐含的意思为zookeeper通信异常
    return false;
}
```

```java
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception
{
    //参数初始化，此处省略
    //...
   
    //自旋获取锁
    while ( !isDone )
    {
        isDone = true;

        try
        {
            //在锁空间下创建临时且有序的子节点
            ourPath = driver.createsTheLock(client, path, localLockNodeBytes);
            //判断是否获得锁（子节点序号最小），获得锁则直接返回，否则阻塞等待前一个子节点删除通知
            hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
        }
        catch ( KeeperException.NoNodeException e )
        {
            //对于NoNodeException，代码中确保了只有发生session过期才会在这里抛出NoNodeException，因此这里根据重试策略进行重试
            if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) )
            {
                isDone = false;
            }
            else
            {
                throw e;
            }
        }
    }

    //如果获得锁则返回该子节点的路径
    if ( hasTheLock )
    {
        return ourPath;
    }

    return null;
}
```

看下internalLockLoop是怎么判断锁以及阻塞等待的，这里删除了一些无关代码，只保留主流程：
```java
//自旋直至获得锁
while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock )
{
    //获取所有的子节点列表，并且按序号从小到大排序
    List<String>        children = getSortedChildren();
    
    //根据序号判断当前子节点是否为最小子节点
    String              sequenceNodeName = ourPath.substring(basePath.length() + 1); // +1 to include the slash
    PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
    if ( predicateResults.getsTheLock() )
    {
        //如果为最小子节点则认为获得锁
        haveTheLock = true;
    }
    else
    {
        //否则获取前一个子节点
        String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

        //这里使用对象监视器做线程同步，当获取不到锁时监听前一个子节点删除消息并且进行wait()，当前一个子节点删除（也就是锁释放）时，回调会通过notifyAll唤醒此线程，此线程继续自旋判断是否获得锁
        synchronized(this)
        {
            try 
            {
                //这里使用getData()接口而不是checkExists()是因为，如果前一个子节点已经被删除了那么会抛出异常而且不会设置事件监听器，而checkExists虽然也可以获取到节点是否存在的信息但是同时设置了监听器，这个监听器其实永远不会触发，对于zookeeper来说属于资源泄露
                client.getData().usingWatcher(watcher).forPath(previousSequencePath);

                //如果设置了阻塞等待的时间
                if ( millisToWait != null )
                {
                    millisToWait -= (System.currentTimeMillis() - startMillis);
                    startMillis = System.currentTimeMillis();
                    if ( millisToWait <= 0 )
                    {
                        doDelete = true;    // 等待时间到达，删除对应的子节点
                        break;
                    }
                    
                    //等待相应的时间
                    wait(millisToWait);
                }
                else
                {
                   //永远等待
                    wait();
                }
            }
            catch ( KeeperException.NoNodeException e ) 
            {
                //上面使用getData来设置监听器时，如果前一个子节点已经被删除那么会抛出NoNodeException，只需要自旋一次即可，无需额外处理
            }
        }
    }
}
```
代码中设置的事件监听器，在事件发生回调时只是简单的notifyAll唤醒当前线程以重新自旋判断.
## 关于反应式编程的思想：
反应式编程框架主要采用了观察者模式，而SpringReactor的核心则是对观察者模式的一种衍伸。关于观察者模式的架构中被观察者(Observable)和观察者(Subscriber)处在不同的线程环境中时，由于者各自的工作量不一样，导致它们产生事件和处理事件的速度不一样，这时就出现了两种情况：
- 1.被观察者产生事件慢一些，观察者处理事件很快。那么观察者就会等着被观察者发送事件，（好比观察者在等米下锅，程序等待，这没有问题）。
- 2.被观察者产生事件的速度很快，而观察者处理很慢。那就出问题了，如果不作处理的话，事件会堆积起来，最终挤爆你的内存，导致程序崩溃。（好比被观察者生产的大米没人吃，堆积最后就会烂掉）。**为了方便下面理解Mono和Flux，也可以理解为Publisher（发布者也可以理解为被观察者）主动推送数据给Subscriber（订阅者也可以叫观察者），如果Publisher发布消息太快，超过了Subscriber的处理速度，如何处理。这时就出现了Backpressure**（背压-----指在异步场景中，被观察者发送事件速度远快于观察者的处理速度的情况下，一种告诉上游的被观察者降低发送速度的策略）

> ```Mono和Flux都是Publisher（发布者）```

Reactor的主要类：
在Reactor中，经常使用的类并不多，主要有以下两个：
- Mono 实现了 org.reactivestreams.Publisher 接口，代表0到1个元素的发布者（Publisher）。
- Flux 同样实现了 org.reactivestreams.Publisher 接口，代表0到N个元素的发布者（Subscriber）。
可能会使用到的类：

Scheduler 表示背后驱动反应式流的调度器，通常由各种线程池实现。

## Reactor带来的优势：
- 使用reactor: 不阻塞主线程，把比较重的逻辑或者IO放到专门的后面处理的线程池处理。这样主线程不阻塞，可以增大吞吐量，提升qps.
- 不使用reactor: 来一个请求，阻塞主线程，那么主线程接收的请求量就有限，吞吐量就降下来了。
- 非阻塞应用程序借助少量的硬件就能提供很好的性能和吞吐量。通过限制线程的数量，我们能够充分利用CPU资源，而不会消耗千兆字节的内存。
- 利用更少的线程做更多的事。
- “回压”机制使得订阅者可以无限接受数据并让它的源头“满负荷”推送所有的数据，也可以通过使用request方法来告知源头它一次最多能够处理 n 个元素，从而将“推送”模式转换为“推送+拉取”混合的模式。

## 切换调度器的操作符
Reactor 提供了两种在响应式链中调整调度器 Scheduler的方法：```publishOn```和```subscribeOn```。它们都接受一个 ```Scheduler```作为参数，从而可以改变调度器。但是```publishOn```在链中出现的位置是有讲究的，而```subscribeOn ```则无所谓。

<img src="../../imgs/reactor线程切换.png">

```java
// 假设与上图对应的代码是：
Flux.range(1, 1000)
.map(…)
.publishOn(Schedulers.elastic()).filter(…)
.publishOn(Schedulers.parallel()).flatMap(…)
.subscribeOn(Schedulers.single())
```
- 如图所示，publishOn会影响链中其后的操作符，比如第一个publishOn调整调度器为elastic，则filter的处理操作是在弹性线程池中执行的；同理，flatMap是执行在固定大小的parallel线程池中的；
- ```subscribeOn无论出现在什么位置，都只影响源头的执行环境，也就是range方法是执行在单线程中的，直至被第一个publishOn切换调度器之前，所以range后的map也在单线程中执行。```**RxJava的线程切换策略与reactor相同。**

## 与CompletableFuture的不同：
- Observable可以像Java 8中的CompletableFuture（一样是非阻塞的，但是与后者不同，Observable还是延迟执行的。除非进行订阅，否则设计良好的Observable不会执行任何操作。
- RxJava的一个显著特征就是声明式并发，而不是命令式并发。手动创建和管理线程已经是过去的事情了。
- 它是hot类型的。
    - CompletableFuture的后台计算是立即启动的，无论是否有人注册thenApply()这样的回调.  
- 它的结果是缓存的。
    - CompletableFuture的后台计算会立即触发，得到的结果会被转发至所有注册的回调。除此之外，如果有的回调是在CompletableFuture完成之后注册的，这个回调会立即基于完成时的值（或异常）进行调用。
- 它只能发布一个值或异常。
    - 理论上，Future<T>只能完成一次（或者永远不进入完成状态），并且带有T类型的返回值或异常。这符合Observable的契约。

## IO流阻塞的问题：
> 在Java中，扩展性的一个限制就是其I/O机制。java.io包设计得非常好，包含了很多小型的Input/OutputStream和Reader/Writer实现，它们互相修饰和包装，每次只添加一项功能。尽管我非常喜欢这种优雅的关注点分离，但是在Java中标准I/O完全是阻塞的，这意味着每个想通过Socket或File进行读取和写入的线程必须无限期地等待结果。更糟糕的是，由于网络速度或磁盘旋转速度缓慢，线程卡顿在I/O操作中很难中断。```阻塞本身不是问题，一个线程被阻塞时，其他线程仍然可以与其他打开的Socket交互。但是创建和管理线程的成本很高昂，而且线程之间的切换也需要时间。```Java应用程序完全能够处理数万个并发连接，但必须非常仔细地进行设计。RxJava与一些事件驱动的现代库协作时，这种设计工作就能大幅减少。

## 选择恰当的并发抽象
在RxJava中，与CompletableFuture最接近的就是Single了。我们也可以使用Observable，不过需要注意它会发布任意数量的值。future和RxJava类型有一个很大的区别，即后者是延迟执行的。得到一个CompletableFuture引用时，我们就可以确定后台计算已经开始了，而Single和Observable很可能在订阅之后才会开始工作。了解了这种语义上的差异，我们就可以很容易地交替使用CompletableFuture、Observable。  
在某些罕见情况下，无法获取异步计算的结果，或者结果本身无关紧要，那么可以使用CompletableFuture<Void>或Observable<Void>。前者比较简单，而后者可能暗示着一个空事件形成的潜在无穷流。rx.Single<Void>是一种糟糕的用法，与Future中返回的Void类似。因此，RxJava引入了rx.Completable。如果我们的架构有很多操作，却不会形成有意义的结果（不过可能会有异常），那么可以考虑使用Completable。举例来说，在命令查询分离模式架构（Command-Query Separation,CQS）中，命令是异步的，而且按照定义它们不会返回任何结果。


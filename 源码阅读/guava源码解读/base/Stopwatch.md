### API测试
```java
/**
    * 测试Stopwatch常见API
    */
@Test
public void test07() throws InterruptedException {
    Stopwatch stopwatch = Stopwatch.createStarted();
    Thread.sleep(2000);
    stopwatch.stop(); // optional
    long millis = stopwatch.elapsed(TimeUnit.SECONDS);
    System.out.println(millis);

}
运行结果：
2
```
<div data-note-content="" class="show-content">
          <div class="show-content-free">
            <h1>问题</h1>
<p>一直在使用如下代码进行程序耗时计算和性能调试，但是对其返回值代表的具体意义却不甚了解。</p>
<p>查看源码发现其代码并不复杂，下面就对其源码进行剖析和解释。</p>
<h1>源码剖析</h1>
<pre class="hljs java"><code class="java"><span class="hljs-keyword">public</span> <span class="hljs-keyword">final</span> <span class="hljs-class"><span class="hljs-keyword">class</span> <span class="hljs-title">Stopwatch</span> </span>{
  <span class="hljs-keyword">private</span> <span class="hljs-keyword">final</span> Ticker ticker;<span class="hljs-comment">//计时器，用于获取当前时间</span>
  <span class="hljs-keyword">private</span> <span class="hljs-keyword">boolean</span> isRunning;<span class="hljs-comment">//计时器是否运行中的状态标记</span>
  <span class="hljs-keyword">private</span> <span class="hljs-keyword">long</span> elapsedNanos;<span class="hljs-comment">//用于标记从计时器开启到调用统计的方法时过去的时间</span>
  <span class="hljs-keyword">private</span> <span class="hljs-keyword">long</span> startTick;<span class="hljs-comment">//计时器开启的时刻时间</span>
}
</code></pre>
<p>通过对elapsedNanos、startTick结合当前时刻时间，可以计算出我们所需要的程序运行流逝的时间长度。</p>
<p>首先来看Ticker工具类：</p>
<pre class="hljs java"><code class="java"><span class="hljs-function"><span class="hljs java">public</span> <span class="hljs-keyword">static</span> Stopwatch <span class="hljs-title">createStarted</span><span class="hljs-params">()</span> </span>{
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">new</span> Stopwatch().start();
}

Stopwatch() {
    <span class="hljs-keyword">this</span>.ticker = Ticker.systemTicker();
  }


<span class="hljs java">private</span> <span class="hljs-keyword">static</span> <span class="hljs java">final</span> Ticker SYSTEM_TICKER =
      <span class="hljs java">new</span> Ticker() {
        <span class="hljs-meta">@Override</span>
        <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">long</span> <span class="hljs-title">read</span><span class="hljs-params">()</span> </span>{
          <span class="hljs-comment">//ticker工具类read方法，直接获取机器的毫秒时间</span>
          <span class="hljs-keyword">return</span> Platform.systemNanoTime();
        }
      };


<span class="hljs-function"><span class="hljs-keyword">static</span> <span class="hljs-keyword">long</span> <span class="hljs-title">systemNanoTime</span><span class="hljs-params">()</span> </span>{
    <span class="hljs-keyword">return</span> System.nanoTime();
  }
</code></pre>
<p>StopWatch的几个关键方法：</p>
<pre class="hljs java"><code class="java"><span class="hljs-function"><span class="hljs-keyword">public</span> Stopwatch <span class="hljs-title">start</span><span class="hljs-params">()</span> </span>{
    checkState(!isRunning, <span class="hljs-string">"This stopwatch is already running."</span>);
    isRunning = <span class="hljs-keyword">true</span>;
    startTick = ticker.read();<span class="hljs-comment">//设置startTick时间为stopwatch开始启动的时刻时间</span>
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">this</span>;
  }
</code></pre>
<pre class="hljs java"><code class="java"><span class="hljs-function"><span class="hljs-keyword">public</span> Stopwatch <span class="hljs-title">stop</span><span class="hljs-params">()</span> </span>{
    <span class="hljs-keyword">long</span> tick = ticker.read();
    checkState(isRunning, <span class="hljs-string">"This stopwatch is already stopped."</span>);
    isRunning = <span class="hljs-keyword">false</span>;
    <span class="hljs-comment">//设置elapsedNanos时间为方法调用时间-stopwatch开启时间+上次程序stopwatch的elapsedNanos历史时间 </span>
    elapsedNanos += tick - startTick;
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">this</span>;
  }

</code></pre>
<pre class="hljs java"><code class="java"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">long</span> <span class="hljs-title">elapsed</span><span class="hljs-params">(TimeUnit desiredUnit)</span> </span>{
    <span class="hljs-keyword">return</span> desiredUnit.convert(elapsedNanos(), NANOSECONDS);
  }


<span class="hljs-function"><span class="hljs-keyword">private</span> <span class="hljs-keyword">long</span> <span class="hljs-title">elapsedNanos</span><span class="hljs-params">()</span> </span>{
    <span class="hljs-comment">//如果stopwatch仍在运行中，返回当前时刻时间-stopwatch开启时刻时间+历史elapsedNanos时间（elapsedNanos只在stop和reset时会更新）</span>
    <span class="hljs-comment">//如果stopwatch已停止运行，则直接返回elapsedNanos，详见stop()</span>
    <span class="hljs-keyword">return</span> isRunning ? ticker.read() - startTick + elapsedNanos : elapsedNanos;
  }
</code></pre>
<h1>结论</h1>
<p>调用方式1：</p>
<pre class="hljs java"><code class="java">Stopwatch stopwatch = Stopwatch.createStarted();
doSomething();
stopwatch.stop(); <span class="hljs-comment">// optional</span>
<span class="hljs-keyword">long</span> millis = stopwatch.elapsed(MILLISECONDS);
log.info(<span class="hljs-string">"time: "</span> + stopwatch); <span class="hljs-comment">// formatted string like "12.3 ms"</span>
</code></pre>
<p>使用stopwatch对程序运行时间进行调试，首先调用<code>StopWatch.createStarted()</code>创建并启动一个stopwatch实例，调用stopwatch.stop()停止计时，此时会更新stopwatch的elapsedNanos时间，为stopwatch开始启动到结束计时的时间，再次调用stopwatch.elapsed()，获取stopwatch在start-stop时间段，时间流逝的长度。</p>
<p>调用方式2：</p>
<pre class="hljs java"><code class="java">Stopwatch stopwatch = Stopwatch.createStarted();
doSomething();
<span class="hljs-comment">//stopwatch.stop();</span>
<span class="hljs-keyword">long</span> millis = stopwatch.elapsed(MILLISECONDS);
log.info(<span class="hljs-string">"time: "</span> + stopwatch); <span class="hljs-comment">// formatted string like "12.3 ms"</span>
</code></pre>
<p>createStarted启动了一个stopwatch实例，stopwatch的时间持续流逝，调用elapsed方法，返回<code>isRunning ? ticker.read() - startTick + elapsedNanos : elapsedNanos;</code>，此时得到的返回值是当前时间和stopwatch.start()时刻时间的时间差值，所以是一个持续递增的时间。</p>
<blockquote>
<p>如果需要在程序中对关键步骤的每一步进行进行持续度量，需要使用如下调用方式</p>
</blockquote>
<pre class="hljs java"><code class="java">Stopwatch stopwatch = Stopwatch.createStarted();
doSomething();
stopwatch.stop();
<span class="hljs-keyword">long</span> millis = stopwatch.elapsed(MILLISECONDS);
log.info(<span class="hljs-string">"time: "</span> + stopwatch); <span class="hljs-comment">// formatted string like "12.3 ms"</span>

stopwatch.reset().start();
doSomething();
stopwatch.stop();
<span class="hljs-keyword">long</span> millis = stopwatch.elapsed(MILLISECONDS);
log.info(<span class="hljs-string">"time: "</span> + stopwatch); <span class="hljs-comment">// formatted string like "12.3 ms"</span>
</code></pre>

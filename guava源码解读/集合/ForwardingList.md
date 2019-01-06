### 测试样例
```java
package 常用的工具类.guava;

import com.google.common.collect.ForwardingList;
import lombok.extern.slf4j.Slf4j;

import java.util.List;

/**
 * @author wxi.wang
 * 18-12-22
 * 测试ForwardingList方法
 */
@Slf4j
public class MyArrayList<E> extends ForwardingList<E> {
    List<E> delegate;

    /**
     * 只能使用构造函数来初始化代理容器,这样每次返回的都是同一个容器
     */
    public MyArrayList(List<E> list) {
        this.delegate = list;
    }

    @Override
    protected List<E> delegate() {
        /**
         * 需要返回当前的list容器,不能在这里new ArrayList<>(),,因为你每次调用ForwardingList里面的
         *  size(),add()方法时,都是调用delegate().add(element);delegate().size().
         *  注意上面的这个delegate().size()方法,每次都需要调用delegate()这个方法,
         *  如果在此方法里面new ArrayList的话,这样每次都创建一个新的容器.. 就不会有size()了...添加一个元素,它的size()并没有变
         */
        return delegate;
    }

    @Override
    public boolean add(E element) {
        log.info("ssssssssssssssssss11");
        return super.add(element);
    }

}
/**
     * 测试ForwardingList
     */
@Test
public void test11() {
    MyArrayList<Integer> a = new MyArrayList<>(new ArrayList<>());
    a.add(11);
    a.add(12);
    a.add(13);
    a.add(14);
    System.out.println(a);
    System.out.println(a.size());
    // 需要在外面定义,保证每次返回的都是同一个容器
    Multimap<String, Integer> multimap = HashMultimap.create();
    ForwardingMultimap forwardingMultimap = new ForwardingMultimap() {
        @Override
        protected Multimap delegate() {
            return multimap;
        }
    };
    forwardingMultimap.put("aa", 1);
    forwardingMultimap.put("aa", 2);
    System.out.println(forwardingMultimap);
}

输出结果:
19:50:57.360 [main] INFO 常用的工具类.guava.MyArrayList - ssssssssssssssssss11
19:50:57.365 [main] INFO 常用的工具类.guava.MyArrayList - ssssssssssssssssss11
19:50:57.365 [main] INFO 常用的工具类.guava.MyArrayList - ssssssssssssssssss11
19:50:57.365 [main] INFO 常用的工具类.guava.MyArrayList - ssssssssssssssssss11
[11, 12, 13, 14]
4
{aa=[1, 2]}
```

### Forwarding装饰器
针对所有类型的集合接口，Guava都提供了Forwarding抽象类以简化装饰者模式的使用。

Forwarding抽象类定义了一个抽象方法：delegate()，你可以覆盖这个方法来返回被装饰对象。所有其他方法都会直接委托给delegate()。例如说：ForwardingList.get(int)实际上执行了delegate().get(int)。

通过创建ForwardingXXX的子类并实现delegate()方法，可以选择性地覆盖子类的方法来增加装饰功能，而不需要自己委托每个方法——译者注：因为所有方法都默认委托给delegate()返回的对象，你可以只覆盖需要装饰的方法。

此外，很多集合方法都对应一个”标准方法[standardxxx]”实现，可以用来恢复被装饰对象的默认行为，以提供相同的优点。比如在扩展AbstractList或JDK中的其他骨架类时，可以使用类似standardAddAll这样的方法。

让我们看看这个例子。假定你想装饰一个List，让其记录所有添加进来的元素。当然，无论元素是用什么方法——add(int, E), add(E), 或addAll(Collection)——添加进来的，我们都希望进行记录，因此我们需要覆盖所有这些方法。
```java

class AddLoggingList<E> extends ForwardingList<E> { 
    final List<E> delegate; // backing list 

    //还需要添加一个构造函数 来实例化 delegate.
    public AddLoggingList(List<E> list) {
        this.delegate = list;
    }

    @Override 
    protected List<E> delegate() { 
        return delegate; 
    } 
    @Override 
    public void add(int index, E elem) { 
        log(index, elem); 
        super.add(index, elem); 
    } 
    @Override 
    public boolean add(E elem) { 
        return standardAdd(elem); // implements in terms of add(int, E) 
    } 
    @Override 
    public boolean addAll(Collection<? extends E> c) { 
        return standardAddAll(c); // implements in terms of add 
    } 
}
```
记住，默认情况下，所有方法都直接转发到被代理对象，因此覆盖ForwardingMap.put并不会改变ForwardingMap.putAll的行为。小心覆盖所有需要改变行为的方法，并且确保装饰后的集合满足接口契约。

通常来说，类似于AbstractList的抽象集合骨架类，其大多数方法在Forwarding装饰器中都有对应的”标准方法”实现。

对提供特定视图的接口，Forwarding装饰器也为这些视图提供了相应的”标准方法”实现。例如，ForwardingMap提供StandardKeySet、StandardValues和StandardEntrySet类，它们在可以的情况下都会把自己的方法委托给被装饰的Map，把不能委托的声明为抽象方法。

<table><thead><tr><th>Interface</th>
			<th>Forwarding Decorator</th>
		</tr></thead><tbody><tr><td><code>Collection</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingCollection.html" rel="nofollow" target="_blank"><code>ForwardingCollection</code></a></td>
		</tr><tr><td><code>List</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingList.html" rel="nofollow" target="_blank"><code>ForwardingList</code></a></td>
		</tr><tr><td><code>Set</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingSet.html" rel="nofollow" target="_blank"><code>ForwardingSet</code></a></td>
		</tr><tr><td><code>SortedSet</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingSortedSet.html" rel="nofollow" target="_blank"><code>ForwardingSortedSet</code></a></td>
		</tr><tr><td><code>Map</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingMap.html" rel="nofollow" target="_blank"><code>ForwardingMap</code></a></td>
		</tr><tr><td><code>SortedMap</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingSortedMap.html" rel="nofollow" target="_blank"><code>ForwardingSortedMap</code></a></td>
		</tr><tr><td><code>ConcurrentMap</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingConcurrentMap.html" rel="nofollow" target="_blank"><code>ForwardingConcurrentMap</code></a></td>
		</tr><tr><td><code>Map.Entry</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingMapEntry.html" rel="nofollow" target="_blank"><code>ForwardingMapEntry</code></a></td>
		</tr><tr><td><code>Queue</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingQueue.html" rel="nofollow" target="_blank"><code>ForwardingQueue</code></a></td>
		</tr><tr><td><code>Iterator</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingIterator.html" rel="nofollow" target="_blank"><code>ForwardingIterator</code></a></td>
		</tr><tr><td><code>ListIterator</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingListIterator.html" rel="nofollow" target="_blank"><code>ForwardingListIterator</code></a></td>
		</tr><tr><td><code>Multiset</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingMultiset.html" rel="nofollow" target="_blank"><code>ForwardingMultiset</code></a></td>
		</tr><tr><td><code>Multimap</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingMultimap.html" rel="nofollow" target="_blank"><code>ForwardingMultimap</code></a></td>
		</tr><tr><td><code>ListMultimap</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingListMultimap.html" rel="nofollow" target="_blank"><code>ForwardingListMultimap</code></a></td>
		</tr><tr><td><code>SetMultimap</code></td>
			<td><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ForwardingSetMultimap.html" rel="nofollow" target="_blank"><code>ForwardingSetMultimap</code></a></td>
		</tr></tbody></table>
```附上官方文档网址：```<a href="https://github.com/google/guava/wiki/NewCollectionTypesExplained">New Collection Types</a>
### 0、首先介绍下这部分集合的特点：  
```These are all designed to coexist happily with the JDK collections framework.```
## １、从整体上看Multiset继承结构：

```java
public interface Multiset<E> extends Collection<E> {
```
```Multiset```继承自jdk的Collection接口，而Collection接口又继承了集合的顶层接口Iterable。而实现Collection又有我们熟悉的List,AbstractList,ArrayList等。  
所以在文档中有这句话：```Notably, Multiset is fully consistent with the contract of the Collection interface```.  
### 1.1、HashMultiset的源码分析  

```Multiset is a true Collection type```  

**```HashMultiset```的继承结构如下：**  
从HashMultiset往上看：
```java
public final class HashMultiset<E> extends AbstractMapBasedMultiset<E>
abstract class AbstractMapBasedMultiset<E> extends AbstractMultiset<E> implements Serializable 
AbstractMultiset<E> extends AbstractCollection<E> implements Multiset<E>
```
不知道大家发现没有，HashMultiset的继承实现方式和JDK中ArrayList的继承实现方式很接近。HashMultiset也并不是直接实现了Multiset接口，而是中间利用了一层```AbstractMultiset```这个作为过滤，在JDK中也有```AbstractList```作为中间过滤。在```《Effective Java》```一书中，讲到为什么要这样设计：原话我忘了，大致的意思是，把List中的通用方法在AbstractList中实现一部分，其余的方法让子类自己实现。这种实现方式更加易于功能的扩展。  
**```HashMultiset```的性能与时间复杂度分析：**   
> The memory consumption of Multiset implementations is linear in the number of distinct elements. 内存消耗为线性，空间复杂度为O(n).  
For a HashMultiset, count is O(1), for a TreeMultiset, count is O(log n).前者是hash表，后者是　红黑树实现。  

**```HashMultiset```的特点与常用ＡＰＩ介绍：**    
1、可以把它看成一个```无序```的ArrayList:  
- add(),增加一个元素，增加次数  
- ```size()```:表示的是所有元素的个数，包括重复的元素。如果需要返回不同元素的的set集合:  
  elementSet(),然后调用size()，就会返回HashMultiset中不同元素的个数。  

2、可以看成一个Map<E, Integer>:  
- count(Object) 返回一个元素出现的次数。上面介绍HashMultiset为O(1).
- entrySet() 该方法返回一个Set<Multiset.Entry<E>> which works analogously to the entrySet of a Map.  
**Multiset.Entry介绍**  
```java
        Multiset<Character> set = HashMultiset.create();
        Set<Multiset.Entry<Character>> entries = set.entrySet();
        for (Multiset.Entry e : entries) {
            System.out.println(e.getElement() + "元素出现的次数是：" + e.getCount());
        }
```  
- elementSet() returns a Set<E> of the ```distinct不同的``` elements of the multiset, like keySet() would for a Map.  
- 更多参照文档。  

**```Multiset```与NULL**  
**每一个Multiset的底层存储都对应table左边的JDK结构。**
<table>
<thead>
<tr>
<th align="left">Map</th>
<th align="left">Corresponding Multiset</th>
<th align="left">Supports <code>null</code> elements</th>
</tr>
</thead>
<tbody>
<tr>
<td align="left"><code>HashMap</code></td>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/HashMultiset.html" rel="nofollow"><code>HashMultiset</code></a></td>
<td align="left">Yes</td>
</tr>
<tr>
<td align="left"><code>TreeMap</code></td>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/TreeMultiset.html" rel="nofollow"><code>TreeMultiset</code></a></td>
<td align="left">Yes</td>
</tr>
<tr>
<td align="left"><code>LinkedHashMap</code></td>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/LinkedHashMultiset.html" rel="nofollow"><code>LinkedHashMultiset</code></a></td>
<td align="left">Yes</td>
</tr>
<tr>
<td align="left"><code>ConcurrentHashMap</code></td>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ConcurrentHashMultiset.html" rel="nofollow"><code>ConcurrentHashMultiset</code></a></td>
<td align="left">No</td>
</tr>
<tr>
<td align="left"><code>ImmutableMap</code></td>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ImmutableMultiset.html" rel="nofollow"><code>ImmutableMultiset</code></a></td>
<td align="left">No</td>
</tr>
</tbody>
</table>

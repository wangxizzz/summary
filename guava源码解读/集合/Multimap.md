### Multimap的继承结构如下:   
```java
public interface Multimap<K, V> 
```
Multimap为顶层接口。  
> Multimap provides a wide variety of implementations. You can use it in most places you would have used a Map<K, Collection<V>>.

**```各种　实现如下Map<K, Collection<V>：```**
<table>
<thead>
<tr>
<th align="left">实现</th>
<th align="left">存储结构(key为Map的key)</th>
<th align="left">值(值为集合的次顶层接口Collection)</th>
</tr>
</thead>
<tbody>
<tr>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ArrayListMultimap.html" rel="nofollow"><code>ArrayListMultimap</code></a></td>
<td align="left"><code>HashMap</code></td>
<td align="left"><code>ArrayList</code></td>
</tr>
<tr>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/HashMultimap.html" rel="nofollow"><code>HashMultimap</code></a></td>
<td align="left"><code>HashMap</code></td>
<td align="left"><code>HashSet</code></td>
</tr>
<tr>
<td align="left">
<a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/LinkedListMultimap.html" rel="nofollow"><code>LinkedListMultimap</code></a> <code>*</code>
</td>
<td align="left"><code>LinkedHashMap``*</code></td>
<td align="left"><code>LinkedList``*</code></td>
</tr>
<tr>
<td align="left">
<a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/LinkedHashMultimap.html" rel="nofollow"><code>LinkedHashMultimap</code></a><code>**</code>
</td>
<td align="left"><code>LinkedHashMap</code></td>
<td align="left"><code>LinkedHashSet</code></td>
</tr>
<tr>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/TreeMultimap.html" rel="nofollow"><code>TreeMultimap</code></a></td>
<td align="left"><code>TreeMap</code></td>
<td align="left"><code>TreeSet</code></td>
</tr>
<tr>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ImmutableListMultimap.html" rel="nofollow"><code>ImmutableListMultimap</code></a></td>
<td align="left"><code>ImmutableMap</code></td>
<td align="left"><code>ImmutableList</code></td>
</tr>
<tr>
<td align="left"><a href="http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/ImmutableSetMultimap.html" rel="nofollow"><code>ImmutableSetMultimap</code></a></td>
<td align="left"><code>ImmutableMap</code></td>
<td align="left"><code>ImmutableSet</code></td>
</tr>
</tbody>
</table>  

### 常用ＡＰＩ介绍：
```java
    public static void main(String[] args) {
        // key为　hashMap的key,values是hashSet,会去除重复的value,而且还是无序的
        Multimap<Integer, Integer> map = HashMultimap.create();
        map.put(1,2);
        map.put(1,3);
        map.put(1,4);
        map.put(2,22);
        map.put(2,23);
        map.put(2,23);
        map.put(2,23);
        System.out.println(map);
        Collection<Map.Entry<Integer, Integer>> entries = map.entries();// 获取所有的Map.Entry
        System.out.println(entries);
        Set<Integer> integers = map.keySet();
        System.out.println(integers);
    }
```
```结果如下：```
> {1=[4, 2, 3], 2=[22, 23]}  

>[1=4, 1=2, 1=3, 2=22, 2=23]  

>[1, 2]

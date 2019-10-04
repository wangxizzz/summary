### 测试API
```java
/**
* 测试Sets常用API
*/
@Test
public void test09() {
    Set<Integer> set1 = Sets.newHashSet(1,2,3,4);
    HashSet<Integer> set2 = Sets.newHashSet(2, 4, 5, 6);
    // 求并集
    Sets.SetView<Integer> union = Sets.union(set1, set2);
    System.out.println(union);
    // 求交集
    Sets.SetView<Integer> intersection = Sets.intersection(set1, set2);
    System.out.println(intersection);
    // 在A不在B中
    Sets.SetView<Integer> difference = Sets.difference(set1, set2);
    System.out.println(difference);
    // 传入的是一个回调方法，然后过滤部分值
    Set<Integer> filter = Sets.filter(set1, (i) -> i > 2);
    System.out.println(filter);
}
运行结果：
[1, 2, 3, 4, 5, 6]  
[2, 4]  
[1, 3]  
[3, 4]
```
### 源码分析
1. union
```java
public static <E> SetView<E> union(final Set<? extends E> set1, final Set<? extends E> set2) {
    checkNotNull(set1, "set1");
    checkNotNull(set2, "set2");

    return new SetView<E>() {
      @Override
      public int size() {
        int size = set1.size();
        for (E e : set2) {
          if (!set1.contains(e)) {
            size++;
          }
        }
        return size;
      }

      @Override
      public boolean isEmpty() {
        return set1.isEmpty() && set2.isEmpty();
      }

      @Override
      public UnmodifiableIterator<E> iterator() {
          // 惰性计算
        return new AbstractIterator<E>() {
          final Iterator<? extends E> itr1 = set1.iterator();
          final Iterator<? extends E> itr2 = set2.iterator();

          @Override
          protected E computeNext() {
            if (itr1.hasNext()) {
              return itr1.next();
            }
            while (itr2.hasNext()) {
              E e = itr2.next();
              if (!set1.contains(e)) {
                return e;
              }
            }
            return endOfData();
          }
        };
      }
```
2. intersection:取交集
```java
public static <E> SetView<E> intersection(final Set<E> set1, final Set<?> set2) {
    checkNotNull(set1, "set1");
    checkNotNull(set2, "set2");

    return new SetView<E>() {
      @Override
      public UnmodifiableIterator<E> iterator() {
        return new AbstractIterator<E>() {
          final Iterator<E> itr = set1.iterator();

          @Override
          protected E computeNext() {
            while (itr.hasNext()) {
              E e = itr.next();
              if (set2.contains(e)) {
                return e;
              }
            }
            return endOfData();
          }
        };
      }
```
3. 在A不在B中
```java
@Override
protected E computeNext() {
while (itr.hasNext()) {
    E e = itr.next();
    if (!set2.contains(e)) {
    return e;
    }
}
return endOfData();
}
```
4. filter过滤
```java
if (unfiltered instanceof SortedSet) {
      return filter((SortedSet<E>) unfiltered, predicate);
    }
    if (unfiltered instanceof FilteredSet) {
      // Support clear(), removeAll(), and retainAll() when filtering a filtered
      // collection.
      FilteredSet<E> filtered = (FilteredSet<E>) unfiltered;
      Predicate<E> combinedPredicate = Predicates.<E>and(filtered.predicate, predicate);
      return new FilteredSet<E>((Set<E>) filtered.unfiltered, combinedPredicate);
    }

    return new FilteredSet<E>(checkNotNull(unfiltered), checkNotNull(predicate));
```
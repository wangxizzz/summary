###测试API
```java
/**
* Lists常用API
*/
@Test
public void test08() {
    /**测试reverse()*/
    List<Integer> list = Lists.newArrayList(1,2,3);
    List<Integer> reverse = Lists.reverse(list);
    System.out.println(reverse);
    // 修改会反映的原list上
    reverse.add(100);
    System.out.println(reverse);
    System.out.println(list);
    /**测试partition,把list进行分区*/
    List<List<Integer>> partition = Lists.partition(Arrays.asList(1, 2, 3, 4), 2);
    System.out.println(partition);
    /**测试transform，转换list的元素**/
    List<Integer> transform = Lists.transform(list, i -> i + 1);
    System.out.println(transform);
}
结果如下：
[3, 2, 1]  
[3, 2, 1, 100]  
[100, 1, 2, 3]  
[[1, 2], [3, 4]]  
[101, 2, 3, 4]  
```

###源码分析：
1. reverse方法方法并不是真的把list reverse一下，返回的reverseList的底层原理是原List支持，只是取了反转后的位置元素而已(计算下标之间的关系)，所以，对原先的List的改变会导致reverseList也跟着改变，其中传进来的List有四种类型情况：
```java
/**
   * 返回这个确切的list的逆转后的视图. 例如, {@code
   * Lists.reverse(Arrays.asList(1, 2, 3))} 返回一个list包含 {@code 3, 2, 1}. 改变这个返回的list，会反射到原list上，并且反之亦然。这个返回的lis通过原list而支持所有的可选择的list操作。
   */
public static <T> List<T> reverse(List<T> list) {
  if (list instanceof ImmutableList) {
    //如果是ImmutableList类型，则调用该类型list的reverse()方法获得reversed的list
    return ((ImmutableList<T>) list).reverse();
  } else if (list instanceof ReverseList) {
     //如果是ReverseList<T>类型，之间返回它的内部成员变量forwardList
     return ((ReverseList<T>) list).getForwardList();
  } else if (list instanceof RandomAccess) {
    //如果指定的list是RandomAccess类型的，返回的list也是random-access的，其中              RandomAccessReverseList<T>也是Lists中的静态内部类，继承自ReverseList<T>类
    return new RandomAccessReverseList<>(list);
  } else {
    //返回ReverseList<>类型，list的参数会被赋值给ReverseList的成员变量private final List<T> forwardList;
    return new ReverseList<>(list);
  }
}
```
ReverseList是Lists的静态内部类，它继承自AbstractList抽象List接口，在ReverseList中有个与自己逆向的forwardList类型
```java
private static class ReverseList<T> extends AbstractList<T> {
  private final List<T> forwardList;
  ...
  //由原先的index计算出逆转后对应的index, 如0123,第一个位置index为0，逆转后为(size-1)-index=(4-1)-0=3,所以逆转list对应index为3
   private int reverseIndex(int index) {
      int size = size();
      checkElementIndex(index, size);
      return (size - 1) - index;
    }
  //reversePosition和上面原理类似，主要用于出入删除元素位置
    private int reversePosition(int index) {
      int size = size();
      checkPositionIndex(index, size);
      return size - index;
    }
  //后面是一些AbstractList中方法的实现，包括迭代器等
  ...
}
```
2. partition方法返回连续的subList即子list，每一个子list长度 一样，内部通过list.subList(start, end)实现但最后一个list可能长度不够小一点，输出的List是不可改变的，但是原list的最新状态映射。其中Partition<>是一个继承自AbstractList<List>的静态内部类.
```java
/**
   * Returns consecutive(连续的　英[kənˈsekjətɪv) {@linkplain List#subList(int, int) sublists} of a list, each of the same
   * size (the final list may be smaller). For example, partitioning a list containing {@code [a, b,
   * c, d, e]} with a partition size of 3 yields {@code [[a, b, c], [d, e]]} -- an outer list
   * containing two inner lists of three and two elements, all in the original order.
   *
   * <p>The outer list is unmodifiable, but reflects the latest state of the source list. The inner
   * lists are sublist views of the original list, produced on demand(需求　英[dɪˈmɑ:nd]) using {@link List#subList(int,
   * int)}, and are subject to all the usual caveats about modification as explained in that API.
   *
   * @param list the list to return consecutive sublists of
   * @param size the desired size of each sublist (the last may be smaller)
   * @return a list of consecutive sublists
   * @throws IllegalArgumentException if {@code partitionSize} is nonpositive
   */
  public static <T> List<List<T>> partition(List<T> list, int size) {
    checkNotNull(list);
    checkArgument(size > 0);
    return (list instanceof RandomAccess)
        // 通过Partition来实现原list的partiotion
        ? new RandomAccessPartition<>(list, size)
        : new Partition<>(list, size);
  }

    private static class RandomAccessPartition<T> extends Partition<T> implements RandomAccess {
        RandomAccessPartition(List<T> list, int size) {
        super(list, size);
    }
  }
  private static class Partition<T> extends AbstractList<List<T>> {
    final List<T> list;
    final int size;

    Partition(List<T> list, int size) {
      this.list = list;
      this.size = size;
    }

    @Override
    public List<T> get(int index) {
      checkElementIndex(index, size());
      int start = index * size;
      int end = Math.min(start + size, list.size());
      return list.subList(start, end);
    }

    @Override
    public int size() {
        // list划分，向上取整
      return IntMath.divide(list.size(), size, RoundingMode.CEILING);
    }
```
3. TransformingSequentialList<F, T>是一个继承自AbstractSequentialList并实现了序列化接口的静态内部类，它有两个成员常量，一个是待转化的原始list，一个是Function<? super F, ? extends T> function函数式接口，作用是通过迭代，运用function中的apply函数把list中的每个元素转为需要转化成的元素，因此只是改了迭代方法，并没有转化后填入一个新的list。需要重写函数式接口Function中的apply方法，在此学习借鉴了Function接口的用法精髓
```java
public static <F, T> List<T> transform(
    List<F> fromList, Function<? super F, ? extends T> function) {
  return (fromList instanceof RandomAccess)
      ? new TransformingRandomAccessList<>(fromList, function)
      : new TransformingSequentialList<>(fromList, function);
}
private static class TransformingSequentialList<F, T> extends AbstractSequentialList<T>
    implements Serializable {
  final List<F> fromList;//待转化的List
  final Function<? super F, ? extends T> function;//函数式接口Function

  TransformingSequentialList(List<F> fromList, Function<? super F, ? extends T> function) {
    this.fromList = checkNotNull(fromList);//检查null
    this.function = checkNotNull(function);//检查null
  }

  /**
   * The default implementation inherited is based on iteration and removal of each element which
   * can be overkill. That's why we forward this call directly to the backing list.
   */
  @Override
  public void clear() {
    fromList.clear();
  }

  @Override
  public int size() {
    return fromList.size();//转化后的list的size和原理的List一样
  }

  @Override
  public ListIterator<T> listIterator(final int index) {
    //在迭代方法中通过apply方法转化元素
    return new TransformedListIterator<F, T>(fromList.listIterator(index)) {
      @Override
      T transform(F from) {
        return function.apply(from);
      }
    };
  }

  @Override
  public boolean removeIf(Predicate<? super T> filter) {
    checkNotNull(filter);
    //评估条件是否成立，成立则删掉该元素，函数式接口Predicate的运用
    return fromList.removeIf(element -> filter.test(function.apply(element)));
  }

  private static final long serialVersionUID = 0;
}
```
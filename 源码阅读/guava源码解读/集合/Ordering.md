### API测试
```java
/**
    * 测试Ordering
    */
@Test
public void test10() {
    List<Integer> list = Lists.newArrayList(1,2,20,3,4,5);
    // for java8 users
    list.stream().collect(Comparators.greatest(3, (Integer x, Integer y) -> Integer.compare(x, y)));
    // 需要传入一个Comparator,然后获取ordering
    Ordering<Integer> ordering = new Ordering<Integer>() {
        @Override
        public int compare(@Nullable Integer left, @Nullable Integer right) {
            // 小到大排序
            return Integer.compare(left, right);
        }
    };
    List<Integer> list1 = ordering.greatestOf(list, 3);
    System.out.println(list1);
}
运行结果：
[20, 5, 4]  
```
### 源码分析:
1. greatestOf:首先需要降序排列，然后取前k个元素
```java
 public <E extends T> List<E> greatestOf(Iterable<E> iterable, int k) {
    // 把当前已经小到大的排序序列,reverse,返回this,然后调用leastOf
    return reverse().leastOf(iterable, k);
  }

public <E extends T> List<E> leastOf(Iterable<E> iterable, int k) {
    if (iterable instanceof Collection) {
        // 先转换为Collection,进行排序
      Collection<E> collection = (Collection<E>) iterable;
      // 如果要找的k大于 集合元素的一般,那么直接排序,直接取的前k个元素即可
      if (collection.size() <= 2L * k) {
        // In this case, just dumping the collection to an array and sorting is
        // faster than using the implementation for Iterator, which is
        // specialized for k much smaller than n.

        @SuppressWarnings("unchecked") // c only contains E's and doesn't escape
        E[] array = (E[]) collection.toArray();
        // 对集合中的元素直接排序(升序排序)
        Arrays.sort(array, this);
        if (array.length > k) {
          array = Arrays.copyOf(array, k);
        }
        return Collections.unmodifiableList(Arrays.asList(array));
      }
    }
    // 如果k比较小的时候 ,不需要进行集合排序,浪费性能
    return leastOf(iterable.iterator(), k);
  }

  public <E extends T> List<E> leastOf(Iterator<E> iterator, int k) {
      // 参数检查
    checkNotNull(iterator);
    checkNonnegative(k, "k");

    if (k == 0 || !iterator.hasNext()) {
      return Collections.emptyList();
    } else if (k >= Integer.MAX_VALUE / 2) {
      // k is really large; just do a straightforward sorted-copy-and-sublist
      // 如果k仍然很大,那么就直接排序并且subList()
      ArrayList<E> list = Lists.newArrayList(iterator);
      Collections.sort(list, this);
      if (list.size() > k) {
          // 如果list.size()大于k,那么就去掉集合中k的后半部分元素,返回前半部分的元素集合
        list.subList(k, list.size()).clear();
      }
      // 这里主要是考虑到ArrayList的扩容机制 ,会导致size(真正的元素个数)与elementData.length不一致
      list.trimToSize();
      return Collections.unmodifiableList(list);
    } else {
        // 如果k比较小
        /* 用到一个特殊算法。但如果要找的元素个数超过总数一半，则不用算法，而是直接排序截取，这样更快。算法适用于k远小n的情况。
            算法流程：
            保持一个2k大小的buffer；每次满了时，清掉较大的一半，剩下k位。
            *剪枝优化：维护一个第k小的阈值，大于它的可以直接忽略了
            *清掉一半的方法：快速选择。定一个标志位，比它小的挪到左边，比它大的挪到右边
            时间O(n + k log k) 存储O(k)
            */
      TopKSelector<E> selector = TopKSelector.least(k, this);
      selector.offerAll(iterator);
      return selector.topK();
    }
  }

  private TopKSelector(Comparator<? super T> comparator, int k) {
    this.comparator = checkNotNull(comparator, "comparator");
    this.k = k;
    checkArgument(k >= 0, "k must be nonnegative, was %s", k);
    this.buffer = (T[]) new Object[k * 2];
    this.bufferSize = 0;
    this.threshold = null;
  }

  /**
   * Adds {@code elem} as a candidate for the top {@code k} elements. This operation takes amortized
   * O(1) time.
   */
  public void offer(@Nullable T elem) {
    if (k == 0) {
      return;
    } else if (bufferSize == 0) {
      buffer[0] = elem;
      threshold = elem;
      bufferSize = 1;
    } else if (bufferSize < k) {
      buffer[bufferSize++] = elem;
      if (comparator.compare(elem, threshold) > 0) {
        threshold = elem;
      }
    } else if (comparator.compare(elem, threshold) < 0) {
      // Otherwise, we can ignore elem; we've seen k better elements.
      buffer[bufferSize++] = elem;
      if (bufferSize == 2 * k) {
        trim();
      }
    }
  }
```
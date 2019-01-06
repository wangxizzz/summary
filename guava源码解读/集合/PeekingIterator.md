### 测试代码
```java
/**
    * 测试PeekingIterator
    */
@Test
public void test12() {
    List<Integer> list = Lists.newArrayList(100,1,1,1,2,3,1,3);
    /**
        * 把list元素去重方法另一个list中,,注意这种去重方式,需要先把list排序
        */
    Collections.sort(list);
    // Iterators.peekingIterator可以把jdk中的utl包下的iterator包装成PeekingIterator
    PeekingIterator<Integer> peekingIterator = Iterators.peekingIterator(list.iterator());
    List<Integer> result = Lists.newArrayList();
    while (peekingIterator.hasNext()) {
        int currentElement = peekingIterator.next();
        // 如果下一个元素等于当前元素,那么就去除掉
        while (peekingIterator.hasNext() && peekingIterator.peek().equals(currentElement)) {
            peekingIterator.next();
        }
        result.add(currentElement);
    }
    System.out.println(result);
}
结果如下:
[1, 2, 3, 100]
```

有时候，普通的Iterator接口还不够。

Iterators提供一个Iterators.peekingIterator(Iterator)方法，来把Iterator包装为PeekingIterator，这是Iterator的子类，它能让你事先窥视[peek()]到下一次调用next()返回的元素。

注意：**Iterators.peekingIterator返回的PeekingIterator不支持在peek()操作之后调用remove()排序等方法。**

传统的实现方式需要记录上一个元素，并在特定情况下后退，但这很难处理且容易出错。相较而言，PeekingIterator在理解和使用上就比较直接了
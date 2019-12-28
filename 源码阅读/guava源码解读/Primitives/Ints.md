### 类结构分析：
```java
public final class Ints  // final，因此里面肯地提供static方法
```

### 常见API测试如下：
```java
/**
* 测试Ints的常用ＡＰＩ
*/
@Test
public void test01() {
    // 把int变为byte数组
    byte[] bytes = Ints.toByteArray(100);
    for (byte b : bytes) {
        System.out.print(b + " ");
    }
    // 把byte数组转换成int
    int value = Ints.fromByteArray(bytes);
    System.out.println(value);
    int[] a = {1,2,3,4};
    // 通过连接符把int数组元素连接起来，返回String
    String s = Ints.join("-", a);
    System.out.println(s);
    // 它的功能是将给定的整形数组转换为List
    List<Integer> list = Ints.asList(a);
    System.out.println(list);
    // 按照给定的进制与参数解析为int类型
    int c = Ints.tryParse("F",16);
    System.out.println(c);
}
输出结果如下：
0 0 0 100   
100  
1-2-3-4  
[1, 2, 3, 4]  
15
```  

### 常见方法分析
- toByteArray:将一个整形value转换为大端表示，以byte array返回，array中每个值有4位。
```java
public static byte[] toByteArray(int value) {
    return new byte[] {
      // 转换为byte数组，分别取高８位，高１６位，高２４位，原值
      (byte) (value >> 24), (byte) (value >> 16), (byte) (value >> 8), (byte) value
    };
  }
```
**这个方法的应用场景在哪儿？**
- fromByteArray:toByteArray方法反过来
```java
public static int fromByteArray(byte[] bytes) {
    // 检查参数是否合法，长度不能小于４
    checkArgument(bytes.length >= BYTES, "array too small: %s < %s", bytes.length, BYTES);
    // 取数组的前四个元素
    return fromBytes(bytes[0], bytes[1], bytes[2], bytes[3]);
}
public static int fromBytes(byte b1, byte b2, byte b3, byte b4) {
    return b1 << 24 | (b2 & 0xFF) << 16 | (b3 & 0xFF) << 8 | (b4 & 0xFF);
}
```
- join(String separator, int... array)：通过连接符把int数组元素连接起来，返回String
```java
  // 采用了可变参数，调用方用数组接收
  public static String join(String separator, int... array) {
    checkNotNull(separator);
    if (array.length == 0) {
      return "";
    }

    // 如果知道大概容量，最好指定，防止频繁扩容。
    StringBuilder builder = new StringBuilder(array.length * 5);
    builder.append(array[0]);
    for (int i = 1; i < array.length; i++) {
      builder.append(separator).append(array[i]);
    }
    return builder.toString();
  }
```
- asList:它的功能是将给定的整形数组转换为List<Integer>
```java
public static List<Integer> asList(int... backingArray) {
    if (backingArray.length == 0) {
      return Collections.emptyList();
    }
    // 返回一个IntArrayAsList
    return new IntArrayAsList(backingArray);
}

// 介绍IntArrayAsList
// IntArrayAsList是Ints里面static class 并且继承了AbstractList，这样就可以实现List接口了。这也是继承与多态的体现，感叹JDK设计的巧妙啊
private static class IntArrayAsList extends AbstractList<Integer>
      implements RandomAccess, Serializable {
    // 有３个成员变量，把原始数组进行封装，并且引入了很多操作List的方法。
    final int[] array;　　// 原始数组
    final int start;
    final int end;

    IntArrayAsList(int[] array) {
      this(array, 0, array.length);
    }

    IntArrayAsList(int[] array, int start, int end) {
      this.array = array;
      this.start = start;
      this.end = end;
    }
```
- lexicographicalComparator:会返回一个比较器，按照一个特定的规则来比较两个整型数组的大小.
```java
public static Comparator<int[]> lexicographicalComparator() {
    return LexicographicalComparator.INSTANCE;
  }
  // 采用enum实现了接口，获得接口里面的方法与字段。
  private enum LexicographicalComparator implements Comparator<int[]> {
    INSTANCE;

    @Override
    public int compare(int[] left, int[] right) {
      int minLength = Math.min(left.length, right.length);
      for (int i = 0; i < minLength; i++) {
        int result = Ints.compare(left[i], right[i]);
        if (result != 0) {
          return result;
        }
      }
      return left.length - right.length;
    }

    @Override
    public String toString() {
      return "Ints.lexicographicalComparator()";
    }
  }
  // 另外Ints里面的compare方法在java7后被标记为deprecated，建议使用Integer.compare这个：
  public static int compare(int x, int y) {
        return (x < y) ? -1 : ((x == y) ? 0 : 1);
  }
```
这个方法采用了枚举：  
这里引出了关于枚举使用的tips：  
　　枚举类型可以存储附加的数据和方法  
　　枚举类型可通过接口来定义行为  
　　枚举类型的项行为可通过接口来访问，跟正常的 Java 类无异values() 方法可用来返回接口中的一个数组  
总而言之，你可以像使用普通 Java 类一样来使用枚举类型。  


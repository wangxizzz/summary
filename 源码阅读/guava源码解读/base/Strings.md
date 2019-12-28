### 类的继承结构
>A few extremely useful string utilities: splitting, joining, padding, and more.
```java
public final class Strings
```
### 测试代码如下：
```java
/**
  * 测试Strings常见API
  */
@Test
public void test02() {
    // repeat把字符串重复count次
    String s1 = Strings.repeat("hello", 3);
    System.out.println(s1);
    // 寻找两个字符串的公共前缀和后缀
    String s2 = "abcdefght";
    String s3 = "abcght";
    System.out.println(Strings.commonPrefix(s2, s3));
    System.out.println(Strings.commonSuffix(s2, s3));
}
运行结果如下：  
hellohellohello  
abc  
ght
```
### 源码分析
- public static String repeat(String string, int count):复制字符串,次数为count
```java
public static String repeat(String string, int count) {
    // 检查是否为null,是跑出NullPointerException.否则就返回string
    checkNotNull(string); 
    // 这样判断是否需要复制有点６啊
    if (count <= 1) {
        // 如果小于０，抛异常
      checkArgument(count >= 0, "invalid count: %s", count);
      // 如果为0就返回"",为１就返回原值
      return (count == 0) ? "" : string;
    }
    public static void checkArgument(boolean b, @Nullable String errorMessageTemplate, int p1) {
        if (!b) {
         throw new IllegalArgumentException(lenientFormat(errorMessageTemplate, p1));
        }
    }

    // IF YOU MODIFY THE CODE HERE, you must update StringsRepeatBenchmark
    // 如果你要修改这段代码，你必须更新这个方法的基准测试
    final int len = string.length();
    // 防止int溢出
    final long longSize = (long) len * (long) count;
    final int size = (int) longSize;
    // 仍然判断int溢出
    if (size != longSize) {
      throw new ArrayIndexOutOfBoundsException("Required array size too large: " + longSize);
    }

    final char[] array = new char[size];
    string.getChars(0, len, array, 0);
    int n;
    // logn复制，复制了logn-1次
    for (n = len; n < size - n; n <<= 1) {
      System.arraycopy(array, 0, array, n, n);
    }
    // 复制最后一次
    System.arraycopy(array, 0, array, n, size - n);
    return new String(array);
  }

  // arraycopy签名如下：
    /*注意为native方法，性能要比java原生态要高。
     * @param      src      原数组.
     * @param      srcPos   原数组要开始复制的起始位置.
     * @param      dest     目标数组.
     * @param      destPos  目标数组的起始位置开始拼接.
     * @param      length   需要复制元素的长度.
     */
    public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```
**System中提供了一个native静态方法arraycopy(),可以使用这个方法来实现数组之间的复制。对于一维数组来说，这种复制属性值传递，修改副本不会影响原来的值。对于二维或者一维数组中存放的是对象时，复制结果是一维的引用变量传递给副本的一维数组，修改副本时，会影响原来的数组。**  
- public static String commonPrefix(CharSequence a, CharSequence b);找出两个字符序列的公共前缀。
```java
public static String commonPrefix(CharSequence a, CharSequence b) {
    checkNotNull(a);
    checkNotNull(b);

    int maxPrefixLength = Math.min(a.length(), b.length());
    int p = 0;
    // 从头开始比较
    while (p < maxPrefixLength && a.charAt(p) == b.charAt(p)) {
      p++;
    }
    // 判断最后两个字符是不是合法的。有一个不合法，截取后的字符串就会降低一位。
    if (validSurrogatePairAt(a, p - 1) || validSurrogatePairAt(b, p - 1)) {
      p--;
    }
    // 利用CharSequence来截取p长度的字符序列，toString()返回
    return a.subSequence(0, p).toString();
  }
  @VisibleForTesting
  static boolean validSurrogatePairAt(CharSequence string, int index) {
    return index >= 0
        && index <= (string.length() - 2)
        && Character.isHighSurrogate(string.charAt(index))
        && Character.isLowSurrogate(string.charAt(index + 1));
  }

  // 说下subSequence方法：
  public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
  subSequence是CharSequence接口中的方法，而String实现了CharSequence接口，同时重写了subSequence方法。
  
  public CharSequence subSequence(int beginIndex, int endIndex) {
       // 调用的是String的substring方法。
        return this.substring(beginIndex, endIndex);
  }
```
- public static String commonSuffix(CharSequence a, CharSequence b); 公共后缀。

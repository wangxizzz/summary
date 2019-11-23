## StringBuilder为什么线程不安全？
一个栗子：
```java
public static void main(String[] args) throws InterruptedException {
    StringBuilder buffer = new StringBuilder();
    for (int i = 0; i < 10; i++) {
        new Thread(() -> {
            for (int j = 0; j < 1000; j++) {
                buffer.append("a");
            }
        }).start();
    }
    System.out.println(buffer.length());
    Thread.sleep(1000);
}
// 输出数字小于10000，并且还抛出ArrayIndexOutOfBoundsException
```
### 分析：
```java
/**
* 底层数据存储
* The value is used for character storage.
*/
char[] value;
// The count is the number of characters used.
int count;

// append()方法:
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);  // 1
    str.getChars(0, len, value, count);   // 2
    // 非原子性相加。因此多线程就会导致count数值比预期小。
    count += len;
    return this;
}
// String对象的getChars方法.
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}

// length()方法：
public int length() {
    return count;
}
```
### ArrayIndexOutOfBoundsException 分析：
现假设有a,b两个线程，StringBuilder的buffer中有4个元素，a,b都向里面append字符串f,那么此时两个线程都走到ensureCapacityInternal方法，假设此时buffer的长度为5(正好)，那么都校验通过了，那么此时b阻塞，a执行完了1,2步骤，count=5,此时b恢复，拿到的count=5,然后执行步骤1，进行数组的复制,如下：
```java
// value = {'f'},srcBegin=0,dst表示目标数组,dstBegin=5(此时就会出现数组越界，因为此时目标数组的总长度为5，而拷贝的下标也从5开始，就会抛异常)
System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
// jdk的数据拷贝方法：把原数组value从srcBegin索引拷贝srcEnd-srcBegin长度到目标数组从dstBegin索引开始。
```
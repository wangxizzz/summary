1.**无符号右移“>>>”与有符号右移“>>”与左移操作**
- Java提供了两种右移运算符：“>>” 和">>>"。其中，“>>”被称为有符号右移运算符，“>>>”被称为无符号右移运算符，它们的功能是将参与运算的对象对应的二进制数右移指定的位数。```二者的不同点在于“>>”在执行右移操作时，若参与运算的数字为正数，则在高位补0；若为负数，则在高位补1。而“>>>”则不同，无论参与运算的数字为正数或为负数，在执运算时，都会在高位补0。```
- 此外，需要注意的是，在对char、byte、short等类型的数进行移位操作前，编译器都会自动地将数值转化为int类型，然后才进行移位操作。由于int型变量只占4Byte（32bit），因此当右移地位数超过32bit时，移位运算没有任何意义。```所以，在Java语言中，为了保证移动位数地有效性，以使右移的位数不超过32bit，采取了取余的操作，即a>>n等价于a>>(n%32)。```
- >> “>>” 有符号右移，相当于除2操作，"<<"有符号左移相当于乘2操作。
- >> 与右移运算不同的是，```左移运算没有有符号和无符号左移动，在左移时，移除高位的同时在低位补0。```与右移运算相同的是，当进行左移运算时，如果移动的位数超过了该类型的最大位数，那么编译器会对移动的位数取模，例如对int类型的数移动33位，实际上只移动了33%32=1位。

2.**BitSet基本介绍**
- Bitset，也就是位图。
- 用来判断一个元素是否出现过。

3.**BitSet分析：**
```java
// 构造函数。初始long数组length=1
public BitSet() {
    // BITS_PER_WORD=64
    initWords(BITS_PER_WORD);
    sizeIsSticky = false;
}

private void initWords(int nbits) {
    words = new long[wordIndex(nbits-1) + 1];
}

private static int wordIndex(int bitIndex) {
    // 右移6位  ADDRESS_BITS_PER_WORD=6
    return bitIndex >> ADDRESS_BITS_PER_WORD;
}
```

```java
public void set(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);
    // wordIndex是吧bitIndex右移6位 => 相当于bitIndex为64以内数字全部都映射到words[0]了。相当于words[0]会存储64个数字的值
    int wordIndex = wordIndex(bitIndex);
    // 2倍扩容
    expandTo(wordIndex);
    // 前面计算wordIndex时，会有64个数字存储在同一个word[index]中，因此采用或运算来存储64个数字的值。1L << bitIndex保证了或运算的正确性
    words[wordIndex] |= (1L << bitIndex); // Restores invariants

    checkInvariants();
}
```

```java
public boolean get(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

    checkInvariants();
    // 获取words的索引值。64个数字会打到同一个idnex
    int wordIndex = wordIndex(bitIndex);
    // 利用与运算获取是否有值
    return (wordIndex < wordsInUse)
        && ((words[wordIndex] & (1L << bitIndex)) != 0);
}
```

4.**应用**

（1）有1千万个随机数，随机数的范围在1到1亿之间。现在要求写出一种算法，将1到1亿之间没有在随机数中的数求出来？  
```java
public static void main(String[] args)
{
    Random random=new Random();
    
    List<Integer> list=new ArrayList<>();
    for(int i=0;i<10000000;i++)
    {
        int randomResult=random.nextInt(100000000);
        list.add(randomResult);
    }
    System.out.println("产生的随机数有");
    for(int i=0;i<list.size();i++)
    {
        System.out.println(list.get(i));
    }
    // 相当于直接创建boolean数组，节省空间
    BitSet bitSet=new BitSet(100000000);
    for(int i=0;i<10000000;i++)
    {
        bitSet.set(list.get(i));
    }
    
    System.out.println("0~1亿不在上述随机数中有" + bitSet.size());
    for (int i = 0; i < 100000000; i++)
    {
        if(!bitSet.get(i))
        {
            System.out.println(i);
        }
    }     
}
```

**1.静态内部类：**  

```应用场景：一般针对于静态内部类不需要外部类的资源，而外部类需要使用内部类的时候,在外部类中实现了与内部类中数据的交互。  ```
```java
public class TestOne {
    public static void main(String[] args){
        // 静态内部类的实例化方式
        Outer.Inner one = new Outer.Inner();
        one.fun2();
    }
}

class Outer{
    public static String name = "什么神奇";
    private int age;
    private Inner inner;
    public static void fun(){
        System.out.println("我靠");
    }
    public static class Inner{
        public void fun2(){
            fun();
            System.out.println(name);
        }
    }
}
```

**2.Mysql的子查询与关联查询的区别：**  
子查询：把内层查询结果当作外层查询的比较条件。会有临时表的产生。

示例：
select goods_id,goods_name from goods where goods_id = (select max(goods_id) from goods);

执行子查询时，MYSQL需要创建临时表，查询完毕后再删除这些临时表，所以，子查询的速度会受到一定的影响，这里多了一个创建和销毁临时表的过程。

优化方式：
可以使用连接查询（JOIN）代替子查询，连接查询不需要建立临时表，因此其速度比子查询快。

3.


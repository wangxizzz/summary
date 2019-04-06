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



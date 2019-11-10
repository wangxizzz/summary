1.**int、long、float转String类型：**

- 第一种方法：s=i+"";  //会产生两个String对象
- 第二种方法：s=String.valueOf(i); //直接使用String类的静态方法，只产生一个对象，堆中也会产生一个对象
- 第一种方法：i=Integer.parseInt(s);//直接使用静态方法，不会产生多余的对象，但会抛出异常
- 第二种方法：i=Integer.valueOf(s).intValue();//Integer.valueOf(s) 相当于 new Integer-(Integer.parseInt(s))，也会抛异常，但会多产生一个对象
- **- 性能对比来看：确实是String.valueOf()性能更高。。。**，参考：https://blog.csdn.net/u012050154/article/details/51320638
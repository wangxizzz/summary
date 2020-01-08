1.**int、long、float转String类型：**  
- 第一种方法：s = i + "";   会产生两个String对象
- 第二种方法(推荐)：s = String.valueOf(i);  直接使用String类的静态方法，只产生一个对象


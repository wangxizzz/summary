### 查看内存对象的占用情况

## 基本参数和概念：
参数如下：  
- heap：打印jvm heap的情况  
- histo：打印jvm heap的直方图。其输出信息包括类名，对象数量，对象占用大小。  
- histo：live ：打印每个class的实例数目,内存占用,类全名信息. VM的内部类名字开头会加上前缀”*”. 如果live子参数加上后,只统计活的对象数量.   
- permstat：打印classload和jvm heap长久层的信息. 包含每个classloader的名字,活泼性,地址,父classloader和加载的class数量. 另外,内部String的数量和占用内存数也会打印出来. 
- F 强迫.在pid没有响应的时候使用-dump或者-histo参数. 在这个模式下,live子参数无效. 

从安全点日志看，从Heap Dump开始，整个JVM都是停顿的，考虑到IO（虽是写到Page Cache，但或许会遇到background flush），几G的Heap可能产生几秒的停顿，在生产环境上执行时谨慎再谨慎。

live的选项，实际上是产生一次Full GC来保证只看还存活的对象。有时候也会故意不加live选项，看历史对象。

Dump出来的文件建议用JDK自带的VisualVM或Eclipse的MAT插件打开，对象的大小有两种统计方式：
- 本身大小(Shallow Size)：对象本来的大小。
- 保留大小(Retained Size)： 当前对象大小 + 当前对象直接或间接引用到的对象的大小总和。

看本身大小时，占大头的都是char[] ,byte[]之类的，没什么意思（用jmap -histo:live pid 看的也是本身大小）。所以需要关心的是保留大小比较大的对象，看谁在引用这些char[], byte[]。

(MAT能看的信息更多，但VisualVM胜在JVM自带，用法如下：命令行输入jvisualvm，文件->装入->堆Dump－>检查 -> 查找20保留大小最大的对象，就会触发保留大小的计算，然后就可以类视图里浏览，按保留大小排序了)

**关于gc相关：https://blog.csdn.net/mccand1234/article/details/52078645  需要整理下**

## 实际应用：
**本例的tomcat端口号是 10239**
```bash
ps -ef|grep tomcat --color
```
查看tomcat的pid（多条时取第一个），线上都是发布到tomcat中,因此无法通过jps命令查看。
```bash
sudo jmap -heap 10239
```
查看堆的内存分配情况

```bash
sudo -u tomcat jmap -histo:live 10239 > a.log

# 或者可以这样查看：
查看对象实例数最多的对象，并按降序排序输出：
执行：jmap -histo <pid>|grep alibaba|sort -k 2 -n -r|more
```
查看tomcat的pid=10239的内存中存活的对象信息，并放入a.log文件中，结果按照使用使用大小逆序排列的   
```一般tomcat的目录文件用户是tomcat用户，因此需要通过 sudo -u $user 登录具体的用户，在使用jvm的其他命令也是如此。```  
>> jmap -histo:live 这个命令执行，JVM会先触发gc，然后再统计信息

利用jmap查看的信息参数解读如下：
```
#instance 是对象的实例个数 
#bytes 是总占用的字节数 
class name 对应的就是 Class 文件里的 class 的标识 
B 代表 byte
C 代表 char
D 代表 double
F 代表 float
I 代表 int
J 代表 long
Z 代表 boolean
前边有 [ 代表数组， [I 就相当于 int[]
对象用 [L+ 类名表示
```

下面分析jmap的结果：  
从打印结果可看出，类名中存在[C、[B等内容，只知道它占用了那么大的内存，但不知道由什么对象创建的。下一步需要将其他dump出来，使用内存分析工具进一步明确它是由谁引用的、由什么对象。
 
jmap -histo:live pid>a.log

可以观察heap中所有对象的情况（heap中所有生存的对象的情况）。包括对象数量和所占空间大小。 可以将其保存到文本中去，在一段时间后，使用文本对比工具，可以对比出GC回收了哪些对象。

jmap -histo:live 这个命令执行，JVM会先触发gc，然后再统计信息。

**可以dump 将内存使用的详细情况输出到文件**

jmap -dump:live,format=b,file=a.log pid

说明：内存信息dump到a.log文件中。

 这个命令执行，JVM会将整个heap的信息dump写入到一个文件，heap如果比较大的话，就会导致这个过程比较耗时，并且执行的过程中为了保证dump的信息是可靠的，所以会暂停应用。

**该命令通常用来分析内存泄漏OOM，通常做法是：**

1）首先配置JVM启动参数，让JVM在遇到OutOfMemoryError时自动生成Dump文件
```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path
```
2）然后使用命令
```bash
jmap  -dump:format=b,file=/path/heap.bin 进程ID(tomcat的pid)  
```
如果只dump heap中的存活对象，则加上选项-live。

3）然后使用MAT分析工具，如jhat命令，eclipse的mat插件。  
使用jhat可以参看：jhat -port 5000 heapDump 在浏览器中访问：http://localhost:5000/ 查看详细信息。

## 总结
1.如果程序内存不足或者频繁GC，很有可能存在内存泄露情况，这时候就要借助Java堆Dump查看对象的情况。  
2.要制作堆Dump可以直接使用jvm自带的jmap命令  
3.可以先使用jmap -heap命令查看堆的使用情况，看一下各个堆空间的占用情况。  
4.使用jmap -histo:[live]查看堆内存中的对象的情况。如果有大量对象在持续被引用，并没有被释放掉，那就产生了内存泄露，就要结合代码，把不用的对象释放掉。  
5.也可以使用 jmap -dump:format=b,file=<fileName>命令将堆信息保存到一个文件中，再借助jhat命令查看详细内容  
6.在内存出现泄露、溢出或者其它前提条件下，建议多dump几次内存，把内存文件进行编号归档，便于后续内存整理分析。  
7.在用cms gc的情况下，执行jmap -heap有些时候会导致进程变T，因此强烈建议别执行这个命令，如果想获取内存目前每个区域的使用状况，可通过jstat -gc或jstat -gccapacity来拿到。



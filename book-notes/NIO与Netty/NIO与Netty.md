
1.socket通信的底层调用：send,recv,sendto,recvfrom系统调用。  
参见网址：https://blog.csdn.net/jirryzhang/article/details/53585855  

2.**基本概念：**
- 孤儿进程：  
- 僵尸进程：
- 

3.**TCP与UDP的粘包与拆包：**
- tcp缓冲区：https://www.cnblogs.com/huanxiyun/articles/5381629.html
- **TCP粘包，UDP不存在粘包问题？**
    - ```参考网址(写的很好)：```https://blog.csdn.net/hik_zxw/article/details/48398935  
- 解决TCP粘包：https://blog.csdn.net/zhangxinrun/article/details/6721495
- 书中给出的几种解决办法：
    - 消息长度固定，累计读取到长度总和为LEN的报文后，就认为读取到了一个完整的消息；将计数器置位，重新开始读取下一个数据报。
    - 将回车换行符作为消息结束符，例如FPT协议，这种方式在文本协议中应用比较广泛。
    - 将特殊的分隔符作为消息的结束标志，回车换行符就是一种特殊的结束分隔符。
    - 通过在消息头中定义长度字段来标识消息的总长度。

4.**linux系统函数：**  
- write() 系统调用: 
    - 函数原型：ssize_t write (int fd, const void * buf, size_t count); 
    - 函数说明：write()会把参数buf所指的内存写入count个字节到参数fd所指的文件内。
    - 返回值：如果顺利write()会返回实际写入的字节数（len）。当有错误发生时则返回-1，错误代码存入errno中。
- read()：
    - 函数定义：ssize_t read(int fd, void * buf, size_t count);
    - 函数说明：read()会把参数fd所指的文件传送count 个字节到buf 指针所指的内存中。
    - 返回值：返回值为实际读取到的字节数, 如果返回0, 表示已到达文件尾或是无可读取的数据。若参数count 为0, 则read()不会有作用并返回0。
- 

5.**nettyTCP粘包/拆包解码器** P113
- LineBasedFrameDecoder + StringDecoder : 这两个组合就是按行切换的文本解码器。(\n, \r\n)
- DelimiterBasedFrameDecoder : 自定义特殊的分隔符作为消息的结束标志
- FixedLengthFrameDecoder : 消息长度固定，累计读取到长度总和为LEN的报文后,就认为读到一个消息。

6.**Java对象编解码技术**
- socket : 描述为: IP + 端口号  
- **socket是什么？**
    - 实际上socket是对TCP/IP协议的封装，它的出现只是使得程序员更方便地使用TCP/IP协议栈而已。socket本身并不是协议，它是应用层与TCP/IP协议族通信的中间软件抽象层，是一组调用接口（TCP/IP网络的API函数）。
    - “TCP/IP只是一个协议栈，就像操作系统的运行机制一样，必须要具体实现，同时还要提供对外的操作接口。 
　　这个就像操作系统会提供标准的编程接口，比如win32编程接口一样。 
　　TCP/IP也要提供可供程序员做网络开发所用的接口，这就是Socket编程接口。”
- Java对象编解码技术：当进行远程跨进程服务调用时，需要把被传输的Java对象编码为字节数组或者ByteBuffer对象。而当远程服务读取到字节数组或者ByteBuffer对象时，需要将它解码为发送时的Java对象。

7.**Java NIO中的API详解**
- 参考网址：  
    - https://segmentfault.com/a/1190000005675241 ， NIO的Channel与Buffer详解

**ctrl c , ctrl v时，系统做了什么？**  

系统的复制、粘贴过程实际上是调用的系统中的API函数，(从用户进程空间复制内容到内核空间，然后再复制到另外的用户进程空间) 。拿windows来说，它会调用GetClipboardData、SetClipboardData等等API函数(Clipboard 剪贴板)。 这是单纯的从应用程序来解释，如果是想更深入的理解这些机制，需要内核调试这些API函数，但是总体上来说，都是通过虚拟内存来复制数据完成的。  

复制的原理是将源文件的副本拷贝到内存的一块缓冲区中，也就是我们说的剪贴板。缓冲区容量有限，遇到大文件会分批进行拷贝 ，只不过我们不会察觉。同时不断将缓冲区内的数据写入到目标位置。即完成文件复制。剪切比其多一个步骤，就是在```复制完成后```将源文件删除，即在文件分配表中为源文件标上删除标记。因此剪切比复制要慢。  

**9.reactor模型**：  


10.epoll所带来的优势：  
- 持有更多的文件句柄数（与系统内存有关）；
- 以前是线性扫描整个socket集合，看哪个准备好了，现在是“活跃“的socket主动去掉callback函数，其他idle的socket就不会。
- 利用共享内存mmap加速内存与用户空间消息的传递；

11.网络编程：  
网络编程的基本模型是Client/Server,也就是两个进程之间的相互通信，服务端提供地址，客户端向服务端监听的地址发送数据，通过TCP三次握手建立连接，如果建立成功，就可以进行socket通信了。

12.**Java NIO参考文章**

**Channel的实现:**  
- FileChannel:从文件中读写数据
- DatagramChannel:通过UDP读写网络中的数据
- SocketChannel:通过TCP读写网络中的数据
- ServerSocketChannel:监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel

https://segmentfault.com/a/1190000006824196 ,NIO的Selector详解

https://segmentfault.com/a/1190000005675241 , NIO的Channel与Buffer详解

https://juejin.im/post/5afd4546f265da0b70261e96 ServerSocketChannel与SocketChannel

https://www.open-open.com/lib/view/open1420790598093.html NIO与传统IO的区别和4中监听事件详解

https://www.jianshu.com/p/746fac80edf8   FileChannel与SocketChannel与流的read()是否阻塞问题(文章讲的是正确的)：  
阻塞IO会在read或者write方法处阻塞，直到有流可读或者将流写入操作系统完成，可以通过Channel.configureBlocking(false)设置为非阻塞（注意FileChannel不能切换为非阻塞模式，而套接字通道可以），非阻塞IO不会在read或者write方法或者accept方法处阻塞，而是会立刻返回(在channel中，如果有数据，就读到buffer中，因此不会阻塞)。

**14.Linux的```sendfile```系统调用：**

java中FileChannel 中transferFrom，transferTo底层使用了这个System Operating.
```java
sendfile(socket, file, len);  
```
运行流程如下：  
1、sendfile系统调用，文件数据被copy至内核缓冲区  
2、再从内核缓冲区copy至内核中socket相关的缓冲区  
3、最后再socket相关的缓冲区copy到协议引擎  

参考网址：https://blog.csdn.net/yusiguyuan/article/details/29350351

****
**nio Buffer 中 compact的作用**
- 我们在 write 后，执行 buffer.compact()将没有发出的数据复制到 buffer 的开始位置，posittion = limit-position,limit = capacity,这样在下一次read(buffer)的时候，数据就会继续添加到缓冲的后面了
- 参考网址：https://blog.csdn.net/jiang_bing/article/details/7878390

**mark,position,limit,capacity的关系**
- 0 <= mark <= position <= limit <= capacity

**Java中new的对象一定是放在heap上，JVM可以直接操作这块区域。**

**内核空间和用户空间：**
- 操作系统所在的空间(内存)为内核空间，也有访问底层硬件设备的所有权限，应用进程所在的空间为用户空间。
- 应用进程是无法直接操作硬件设备和执行特权命令，只有通过系统调用，用户态切换到核态，进行访问。比如IO,socket(涉及网卡交互)。
- 参考网址：https://www.cnblogs.com/sparkdev/p/8410350.html

**Socket缓冲区**
- 每个 socket 被创建后，都会分配两个缓冲区，输入缓冲区和输出缓冲区。write()/send() 并不立即向网络中传输数据，而是先将数据写入缓冲区中，再由TCP协议将数据从缓冲区发送到目标机器。一旦将数据写入到缓冲区，函数就可以成功返回，不管它们有没有到达目标机器，也不管它们何时被发送到网络，这些都是TCP协议负责的事情。TCP协议独立于 write()/send() 函数，数据有可能刚被写入缓冲区就发送到网络，也可能在缓冲区中不断积压，多次写入的数据被一次性发送到网络，这取决于当时的网络情况、当前线程是否空闲等诸多因素，不由程序员控制。read()/recv() 函数也是如此，也从输入缓冲区中读取数据，而不是直接从网络中读取。
- https://www.jianshu.com/p/de482fc0e9fb

**Java NIO零拷贝技术**
- sendfile则没有映射,适用于应用进程不需要对读取的数据做任何处理的场景。
- 传统IO，用户空间通过read(),write()系统调用进行用户态与内核态的切换。
- 0拷贝，应用程序通过sendfile系统调用使用zero copy.
- https://www.jianshu.com/p/e76e3580e356 讲述了Linux2.4后的利用scatter与gather0拷贝
- https://juejin.im/post/5c1c532551882579520b1f47
- 参考网址：http://sound2gd.wang/2018/07/24/Java-NIO%E5%88%86%E6%9E%90-11-%E9%9B%B6%E6%8B%B7%E8%B4%9D%E6%8A%80%E6%9C%AF/ ，此里面含有Java使用2中0拷贝的例子。

**Linux2.4版本后的zero copy:**
- https://www.jianshu.com/p/e76e3580e356
    - 讲述了Linux2.4后的利用scatter与gather0拷贝

**内存文件映射：(结合0拷贝来看)**
- 简要理解：磁盘文件与内存建立映射，这块内存相对JVM来说，是堆外内存，Java程序直接操作堆外内存(Buffer中通过address逻辑地址)，就可以反映到操作文件。
- 首先堆外内存，是针对于JVM而言的，也就是不在JVM管控之外的内存，我们称之为堆外内存。 而共享内存，是指内核态和用户态共享一块内存，也就是内核态虚拟地址和用户态虚拟地址指向了同一个物理内存地址。
- 文件仍然是要读到内存中，只是说不是全部文件，相对于传统的IO,mmap少了把数据复制到进程缓冲区与复制回来的操作。
- https://blog.csdn.net/Evankaka/article/details/48464013
- https://blog.csdn.net/maverick1990/article/details/48050975
- https://blog.csdn.net/mg0832058/article/details/5890688

**RandomAccessFile与共享内存**
- RandomAccessFile是一个独立的类，可读可写随机访问，但可以被MapedByteBuffer取代。
- 参考网址：https://blog.csdn.net/akon_vm/article/details/7429245

**每调用线程池的execute()或者submit(),都会起一个线程(重用或者新创建Thread)**

**NIO的适用场景：多客户端连接，消息不是很大时 适用。**

**字符编码：**
- ASCII(American Standard Code Imformation Interchange)
    - 7 bit 来表示一个字符。共有128个字符(0 ~ 127)
- ISO-8859-1:
    - 一个字节(8bit)表示一个字符，可以表示256个字符，相比于ascII多了128个字符。
- GB2312:
    - 以上两种最多只有256个字符，无法适应中文的数量，因此出现了其他编码。
    - 两个字节表示一个汉字
- GBK:
    - 对GB2312的扩展，将一些生僻字加入
- GB18030:
    - 最全的中文编码
- big5:
    - 针对繁体中文
- unicode:
    - 全世界语言编码集
    - 采用2个字节表示一个字符(有争论)
    - 相对于ascII，占用存储空间。
**以上都是字符编码方式**

**针对unicode占用存储容量问题：**  
下面介绍字符存储方式：
- UTF (Unicode Translation Format):
    - 本身是一种存储格式。
    - UTF-8是unicode的一种实现之一，通常通过3个字节表示一个中文，一个英文字符用一个字节表示。
    - utf-8为了节省资源，采用变长编码，编码长度从1个字节到6个字节不等。
    - utf-16是用两个字节来编码所有的字符，utf-32则选择用4个字节来编码。
- 参考网址:
    - https://www.jianshu.com/p/36d20de2a1ee 

****
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
- ```内核空间与用户空间：```调用系统函数相当于系统调用。内核空间和用户空间是严格区分的，可以通过系统调用来访问内核空间，此时从用户态转为了内核态。一个任务（进程）执行系统调用而陷入内核代码中执行时，我们就称进程处于内核运行态（或简称为内核态）。此时处理器处于特权级最高的（0级）内核代码中执行。当进程处于内核态时，执行的内核代码会使用当前进程的内核栈。每个进程都有自己的内核栈。当进程在执行用户自己的代码时，则称其处于用户运行态（用户态）。
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
- remaining():
- flip():
- 

8.**线程池的并发聊天服务器**


**ctrl c , ctrl v时，系统做了什么？**  

系统的复制、粘贴过程实际上是调用的系统中的API函数，(从用户进程空间复制内容到内核空间，然后再复制到另外的用户进程空间) 。拿windows来说，它会调用GetClipboardData、SetClipboardData等等API函数(Clipboard 剪贴板)。 这是单纯的从应用程序来解释，如果是想更深入的理解这些机制，需要内核调试这些API函数，但是总体上来说，都是通过虚拟内存来复制数据完成的。  

复制的原理是将源文件的副本拷贝到内存的一块缓冲区中，也就是我们说的剪贴板。缓冲区容量有限，遇到大文件会分批进行拷贝 ，只不过我们不会察觉。同时不断将缓冲区内的数据写入到目标位置。即完成文件复制。剪切比其多一个步骤，就是在```复制完成后```将源文件删除，即在文件分配表中为源文件标上删除标记。因此剪切比复制要慢。  
参考网址：https://zhidao.baidu.com/question/483819292.html?qbl=relate_question_5

9.reactor模型：  
https://www.cnblogs.com/doit8791/p/7461479.html
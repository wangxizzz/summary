1、**Channel是什么？**  
Netty Channel 和 Java 原生 Socket 对应，而 Netty NIO Channel 和 Java 原生 NIO SocketChannel 对象。

2、**java命令运行单个java文件：**  
需要把 .java文件中的package信息删掉，否则java命令 运行.class文件会报找不到主方法

```bash
# 编译
javac SocketIO.java
# 运行
java SocketIO.class
```

3、**linux下的strace命令：**  
作用： 跟踪OS系统调用
```bash
# 利用java命令运行SocketIO.class文件，把系统调用log记录在当前目录下的out文件夹下，
# 可以通过查看占用最大的文件，即为main文件的系统调用Log
strace -ff -o out java SocketIO.class
```

4、**select, poll, epoll**
- select, poll是一类多路复用器，只不过select有fd限制(1024)，poll没有，利用array存放建立连接的fd.
- epoll: 
    - 首先没有epoll时poll存在的问题：
        - 每次循环都有与内核的数据传输，需要把注册的channel fd传入内核。由内核每次去帮你遍历这些fd，然后返回有事件的fd。因为此时regster获取的fd是放在应用空间
    - epoll的解决：
        - 1、在内核开辟一块空间
        - 2、把fd添加进去（每register一次，就把register获取的fd导入内核里）
        - 3、通过循环方式去，获取哪些fd可以 R/W ，这中间是通过回调函数.
    - 上述epoll的解决步骤，可以同过strace 的main文件log追踪到。
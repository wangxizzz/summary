1.socket通信的底层调用：send,recv,sendto,recvfrom系统调用。  
参见网址：https://blog.csdn.net/jirryzhang/article/details/53585855  

2.**基本概念：**
- 孤儿进程：  
- 僵尸进程：

3.**TCP：**
- tcp缓冲区：https://www.cnblogs.com/huanxiyun/articles/5381629.html
- 

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


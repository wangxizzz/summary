## curl命令介绍：
1.curl --referer Referer_URL target_URL : 参照页(referer)是位于HTTP头的一个字段，用来标识用户是从哪个页面调到当前页面的。
2.curl http://www.baidu.com --cookie-jar baidu.cookie 把访问百度的cookie记录到本地文件。

3.curl -H "Accept-language: en" http://www.baidu.com -H "Host: www.baidu.com" --user-agent "Mozilla/5.0" -H 表示增加Header, 可以增加多个Header. --user-agent表示用户代理。

4.curl -I http://www.baidu.com  -I 参数或者--head表示只打印HTTP头部信息。

## 网络管理:
1.**网关**:一个网络中的设备如果想同另一个网络中的设备进行通信,就需要借助某个同时连接了两个网络的设备。这个特殊的设备被称为网关,它的作用是在不同的网络中转发分组.

2.route -n 可以以数字的形式查看由操作系统维护的路由表

3.**ping** : Ping发送一个ICMP,回声请求消息给目的地并报告是否收到所希望的ICMP echo （ICMP回声应答）。检查网络是否通畅或者网络连接速度的命令.
- ping www.baidu.com -c 2 :  ping 命令发送了2个 echo 分组后就停止发送
- ping发送数据使用ICMP,ICMP协议通过IP协议发送的，IP协议是一种无连接的，不可靠的数据包协议.而TCP协议提供了可靠的连接.

4.host baidu.com : 列出域名的所有IP地址

5.fping : 可以同时ping一组IP地址,而且响应速度非常快.
- a 指定打印出所有活动主机的IP地址
- u 指定打印出所有无法到达的主机
- g 指定从 "IP地址/子网掩码"记法或者"IP地址范围"记法中生成一组IP地址

> fping -a 100.81.138.65 100.81.138.70 100.81.138.712

6.lftp:
- lftp username@ftphost 连接fpt服务器
- get fileName 从ftp服务器下载文件
- put fileName 上传文件
## 磁盘管理
1.du : disk usage
- du -h 表示以K、M、G 列出文件大小。
- du -a dirName 可递归统计该文件夹下文件所占磁盘空间大小.
- du fileName 可以统计文件所占磁盘空间的大小
- du dirName 可以递归统计该文件夹下文件所占磁盘空间大小.
- du -ch dirName 可以统计该文件夹的总大小
- du -s 统计文件夹总大小 -s(summarize)
- du -h --exclude "*.pdf" FILES documents/ 排除document文件夹下所有的pdf文件,统计其他文件
- du -k或-m 以k或m为单位统计文件大小.

2.找出指定目录的最大10个文件
> 方法一　：　du -ah documents/ | sort -nrk 1 | head -n 10  
> 方法二　：　find . -type f -exec du -k {} \; | sort -nrk 1 | head

**du的实际举例：**  
一般是磁盘报警时因为日志多了导致磁盘使用比例超过阙值，一般步骤如下：  
先df -h 查看一下哪块磁盘使用比例过大，同时df -i 查看inode占用比例
假设是我们经常报警的/home/q目录,按照经验一般是/home/q/logs/目录
然后du -sh * 查看当前目录的大小，找到占用空间比较大的目录
然后进入该目录，根据实际情况查找；  
一般该目录下文件和目录都有，如果用du -sh * 目录可能也不太好查找，可以用下面的命令：  
find . -type d | du -h | grep G  
查找占用空间比较大的目录


3.df 提供磁盘可用空间信息

4.time ls 可以知道ls命令执行的时间

5.w/who 可以查看登录用户的信息

## 对系统进程的管理
- -A ：所有的进程均显示出来，与 -e 具有同样的效用；
- -a ： 显示现行终端机下的所有进程，包括其他用户的进程；
- -u ：以用户为主的进程状态 ；
- -x ：通常与 a 这个参数一起使用，可列出较完整信息。

ps -aux 与 ps -ef 的输出信息相同。

1..ps -efo pid,comm,pcpu | head -n 5 可以查看指定参数的进程信息
- -e : 代表every
- -f :　full 显示多列信息
- -o : 可以指定列出的参数

-o 参数可以为:
|参数|描述|
|--|--|
|pcpu|CPU占用率
pid|进程ID
ppid| 父进程ID
pmem |内存使用率
comm |可执行文件名
cmd |简单命令 1
user |启动进程的用户
nice |优先级
time |累计的CPU时间
etime |进程启动后流逝的时间
tty |所关联的TTY设备
euid |有效用户ID
stat |进程状态

2.对ps列出来的信息进行具体的排序:  
可以用 --sort 将 ps 命令的输出根据特定的列进行排序。在参数前加上 + (升序)或 - (降序)
来指定排序方式:  
ps [OPTIONS] --sort -paramter1,+parameter2,parameter3..  
例子: ps -eo comm,pcpu --sort -pcpu | head

3.**线程信息**:
- ps -eLf --sort -nlwp | head 该命令列出了线程数最多的10个进程
    - nlwp 表示进程的线程数量.
- 更多信息可以参考
    - ps --help
        - 然后根据提示选择ps --help t 就可以参看有关线程的信息

4.**which 、 whereis 、 file 、 whatis：**
- which ls 列出ls命令的位置
- echo $PATH 查看环境变量
- whereis ls : 与which类似
- file : 可以查看文件类型
- whatis ls 

5.**netstat命令：**

**用法介绍：**
- netstat -nptl
    - n表示直接使用ip地址，而不通过域名服务器，p表示进程，t表示tcp，l表示监听(listening)
- 

**命令选项参考如下：**
- -a或--all：显示所有连线中的Socket； 
- -A<网络类型>或--<网络类型>：列出该网络类型连线中的相关地址； 
- -c或--continuous：持续列出网络状态； 
- -C或--cache：显示路由器配置的快取信息； 
- -e或--extend：显示网络其他相关信息； 
- -F或--fib：显示FIB； 
- -g或--groups：显示多重广播功能群组组员名单； 
- -h或--help：在线帮助； 
- -i或--interfaces：显示网络界面信息表单； 
- -l或--listening：显示监控中的服务器的Socket； 
- -M或--masquerade：显示伪装的网络连线； 
- -n或--numeric：直接使用ip地址，而不通过域名服务器； 
- -N或--netlink或--symbolic：显示网络硬件外围设备的符号连接名称； 
- -o或--timers：显示计时器； 
- -p或--programs：显示正在使用Socket的程序识别码和程序名称； 
- -r或--route：显示Routing Table； 
- -s或--statistice：显示网络工作信息统计表； 
- -t或--tcp：显示TCP传输协议的连线状况； 
- -u或--udp：显示UDP传输协议的连线状况； 
- -v或--verbose：显示指令执行过程； 
- -V或--version：显示版本信息； 
- -w或--raw：显示RAW传输协议的连线状况； 
- -x或--unix：此参数的效果和指定"-A unix"参数相同； 
- --ip或--inet：此参数的效果和指定"-A inet"参数相同。
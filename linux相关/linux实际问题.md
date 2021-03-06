1.**找出linux磁盘占用情况并找出最大的目录**
- df -h : 
    - 此命令，既可以找出linux磁盘各个挂载点占用情况, disk free。
    - 不要使用 df -ah,    -a 表示递归找出所有文件(整个磁盘文件太多，难扫描)
    - 不要 df -h /   这样是不能找出```整个```磁盘的使用情况，原因不详、
    - df 不支持 -g参数，如果需要排序，直接查看最大磁盘占用情况，那么可以 df -m | sort -nk3,即可以找出 以 M 为单位的排序情况。
    - **df -h ./ 可以查看当前目录的占用情况, 不能使用 du -sh ./ 这个命令在这里得到结果有误。**

2.**找出一个目录下的最大目录或者文件**
-  sudo du -h --max-depth=1|grep "G"
- 或者： du -h -d 1|grep M ./   => 查看当前目录下文件占用大小
    - 错误写法：
        - du -ah --max-depth=1 | sort -nrk1 | head -n 10

3.**找出一个(或多个工程)文件中包含"Exception"的文件**
- 格式：grep -r "{关键字}"  {路径}
    - 例子： grep -r "移位操作" ~/Document/summary

4.**rm -rf文件删除：**
- 实际上，只有当一个文件的引用计数为0（包括硬链接数）的时候，才可能调用unlink删除，只要它不是0，那么就不会被删除。所谓的删除，也不过是文件名到 inode 的链接删除(只是一个链接删除)，只要不被重新写入新的数据，磁盘上的block数据块不会被删除。因此，你会看到，即便删库跑路了，某些数据还是可以恢复的。换句话说，当一个程序打开一个文件的时候（获取到文件描述符），它的引用计数会被+1，rm虽然看似删除了文件，实际上只是会将引用计数减1，但由于引用计数不为0，因此文件不会被删除。
- lsof |grep deleted
    - 可以 找到哪些文件被删除了，但还是被某些进程打开了。
- 所以删除线上日志文件，用rm -rf是无用的。
    - 使用 cat /dev/null > 文件名  即可。
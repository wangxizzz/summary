## git常用命令. 书名为Git:
0.clone项目具体的分支：
> git clone -b 分支名 url

1.centos7下查看隐藏文件：ls -aF

2.删除git配置的全局信息：
git config --unset --global user.name
git config --unset --global user.email

4、git log  查看提交日志信息

5、git rm -rf 强制删除git仓库和本地磁盘的文件。

6、git checkout -b 分支名
快速创建一个分支，并且切换到该分支。

7、git branch -d 分支名
删除一个分支

8、git branch 
列出所有分支列表，其中前面带有*的分支名，为当前head指针指向的分支，也就是当前分支

9、git branch -v 可以查看每个分支的最后一次提交。

10、git branch --merge:可以查看哪些分支已经合并到当前分支了。那么那些已经合并的分支，就可以删掉了

11、git branch --no-merge:可以查看未合并的工作分支，如果此时删掉那些未合并的分支，会删除失败的。
错误信息如下：
$ git branch -d testing
error: The branch 'testing' is not fully merged.
If you are sure you want to delete it, run 'git branch -D testing'.
当然也可以强行删除，丢掉那部分工作。使用 -D参数即可。

12、``origin'' 并无特殊含义
远程仓库名字 origin'', 与分支名字 master'' 一样， 在 Git 中并没有任何特别的含
义一样。 同时 master'' 是当你运行 git init 时默认的起始分支名字， 原因仅仅是它的广泛使
用， origin'' 是当你运行 git clone 时默认的远程仓库名字。 如果你运行 git
clone -o booyah ， 那么你默认的远程分支名字将会是 booyah/master 。

13、git fetch origin : 从远程origin分支中抓取本地仓库没有的数据,但是不会更新本地数据，需要merge。

14、git push 远程仓库名 远程仓库分支名
例如：git push origin serverfix ：推送到远程仓库origin的serverfix分支上

15、git push -u origin serverfix:awesomebranch 来将本地的 serverfix 分支推送到远程仓库上的
awesomebranch 分支。注意：-u参数 这是 --set-upstream 的简写， 该标记会为之后轻松地推送与拉取配置分支

16、git pull 相当于git fetch 和 git merge。

17、在本地删除远程分支 git push origin --delete serverfix  可以在一段有效时间内恢复的。

18、git add --patch : 可以部分暂存文件

19、
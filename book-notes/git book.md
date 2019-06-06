### 参考书籍：《Git》
### 这篇记载的git命令，都是经过真实环境测试可用的。
### 测试以  https://github.com/wangxizzz/gitdemo 仓库为基础进行测试，并且建立了三个分支，分别是master,wangxi01,wangxi02

```在idea中使用git，新创建的文件时红色，执行git add . 后,变为绿色，执行git commit 后，变为无色。```
Changes not staged for commit:  表示还没有到commit阶段，需要add  
Changes to be committed: 表示需要commit

## 已经commit了，如何回退？？？

### 常用命令：
**git add:**
- 把文件提交到暂存区。

**git commit**
- 把文件提交到本地仓库。(.git文件夹下)

**git clone https://github.com/wangxizzz/gitdemo test**  
- 把远程gitdemo仓库克隆下来，并且改名字为test(默认文件夹名字为gitdemo)。
- 初次克隆某个仓库的时候， 工作目录中的所有文件都属于已跟踪文件， 并处于未修改状态。

**git commit -a -m "test"**
- 会同时执行git add . 和 git cgitommit -m "test" 命令
- ```注意：```
    - 如果在git仓库新创建了一个文件，那么此命令不生效,因为untracked 。还是得git add . 然后commit
    - 如果是修改已经是暂存区的文件时，执行此命令生效。

**git rm 删除文件** 
- 删除git仓库的文件，并且也从磁盘删除了(不能ctrl Z 回退文件)。
- 如果直接在idea中删除，也删除了git仓库的文件，但是可以从磁盘中回退文件。  
- -r 递归删除(删除目录), -f 强制删除

**git rm --cached AA.java**
- 我们想把文件从 Git 仓库中删除（ 亦即从暂存区域移除） ， 但仍然希望保留
在当前工作目录中。 换句话说， 你想让文件保留在磁盘， 但是并不想让 Git 继续跟踪。 当你
忘记添加 .gitignore 文件， 不小心把一个很大的日志文件或一堆 .a 这样的编译生成文件
添加到暂存区时， 这一做法尤其有用。

**git mv AA.java BB.java**
- 在 Git 中对文件改名。还得重新commit，不用add。
- 执行这个命令相当于执行了以下命令：
    - mv README.md README   // 先把源文件改名字
    - git rm README.md      // 然后删除掉原始文件
    - git add README        // 把新文件加入暂存区

**git log**
- 默认不用任何参数的话， git log 会按提交时间列出所有的更新， 最近的更新排在最上面。
- git log -p -2
    - 一个常用的选项是 -p ， 用来显示每次提交的内容差异。 你也可以加上 -2 来仅显示最近两
次提交
- git log --stat
    - 正如你所看到的， --stat 选项在每次提交的下面列出额所有被修改过的文件、 有多少文件被
修改了以及被修改过的文件的哪些行被移除或是添加了。
- git log --pretty=oneline
    - 一行输出剪短信息（还有内置参数，比如short,full, fuller）
- git log --pretty=format:"%h - %an, %ar : %s"
    - 可以格式化输出

git log --pretty=format 常用配置：

|选项| 说明|
|--|--|
|%H |提交对象（ commit） 的完整哈希字串|
|%h| 提交对象的简短哈希字串|
|%T| 树对象（ tree） 的完整哈希字串|
|%t| 树对象的简短哈希字串|
|%P| 父对象（ parent） 的完整哈希字串|
|%p| 父对象的简短哈希字串|
|%an| 作者（ author） 的名字|
|%ae| 作者的电子邮件地址|
|%ad| 作者修订日期（ 可以用 --date= 选项定制格式）|
|%ar| 作者修订日期， 按多久以前的方式显示|
|%cn| 提交者(committer)的名字|
|%ce| 提交者的电子邮件地址|
|%cd| 提交日期|
|%cr| 提交日期， 按多久以前的方式显示|
|%s| 提交说明|

**git log 搜索：**
- git log --since=2.weeks     两个星期之内的提交
- git log --since=2.hours       两个小时之内的提交
- git log --since=2.minutes     两分钟之内的提交
- git log -- ./     只列出当前目录的commit记录
- git log --author=wangxizzz

**搜索的参数如下：**
|选项 |说明|
|--|--|
|-(n) |仅显示最近的 n 条提交|
|--since , --after |仅显示指定时间之后的提交。|
|--until , --before |仅显示指定时间之前的提交。|
|--author| 仅显示指定作者相关的提交。|
|--committer| 仅显示指定提交者相关的提交。|
|--grep |仅显示含指定关键字的提交|
|-S |仅显示添加或移除了某个关键字的提交|


**git checkout -- BB.java**
- 丢弃掉对文件的修改。
- **注意：** 只适用于已经修改的文件，但还没放入暂存区(就是没执行git add .),此命令作用不大

****
### 远程仓库的使用：

**git remote -v**
- 会显示需要读写远程仓库使用的 Git 保存的简写与其对应的 URL。

比如这样：
```
$ git remote -v
origin https://github.com/schacon/ticgit (fetch)
origin https://github.com/schacon/ticgit (push)
```

**git remote add shortname url**
- 添加一个新的远程 Git 仓库， 同时指定一个你可以轻松引用的简写
-例如：git remote add pb https://github.com/paulboone/ticgit

****

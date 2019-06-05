### 参考书籍：《Git》
### 这篇记载的git命令，都是经过真实环境测试可用的。
### 测试以  https://github.com/wangxizzz/gitdemo 仓库为基础进行测试，并且建立了三个分支，分别是master,wangxi01,wangxi02

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
    - 如果在git仓库新创建了一个文件，那么此命令不生效。还是得git add . 然后commit
    - 如果是修改已经是暂存区的文件时，执行此命令生效。

**git rm 删除文件**
- 删除git仓库的文件，并且也从磁盘删除了(不能ctrl Z 回退文件)。
- 如果直接在idea中删除，也删除了git仓库的文件，但是可以从磁盘中回退文件。  

****  
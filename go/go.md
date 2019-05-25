1.go的所有参数传递都是值传递。（除非带了指针与引用）

2.大写代表public,小写代表private.(方法和变量都适用)。

3.go get 下载包。  
但是会碰到包下不下来。  
利用gopm 下载无法下载的包，首先安装gopm。（注意：切换到GOPath目录，利用git下载）  
```go
go get -v github.com/gpmgo/gopm （-v 详细的输出信息）
```
下载安装完gopm后，然后利用gopm下载其他的包。利用```gopm help get```查看gopm get 的使用  
然后就可以下载包：eg: gopm get -v -g -u url;

4.**包之间的函数相互引用问题：**  
如果定义的报名不存在，那么编译器就会去下面的三个地方去找。  
```所以：在新建项目时，需要新建一个xrc目录，并把当前项目加入到GOPATH目录，这样在编译时，就会在项目路径下找到引用的包。```
```java
src\tree\treeentry\entry.go:6:2: cannot find package "treefds" in any of:
	C:\Go\src\treefds (from $GOROOT)    // GOROOT,go的安装目录。
	D:\goprojects\demo01\src\treefds (from $GOPATH)   // 项目的gopath路径
	C:\Users\Administrator\go\src\treefds   // 全局gopath路径
```

5.
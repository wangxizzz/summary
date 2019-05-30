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
```所以：在新建项目时，需要新建一个src目录，并把当前项目加入到GOPATH目录,这样在编译时，就会在项目路径下找到引用的包。```  
**去掉使用global gopath的勾，**  
<img src="../imgs/goland.png"/>
```java
src\tree\treeentry\entry.go:6:2: cannot find package "treefds" in any of:
	C:\Go\src\treefds (from $GOROOT)    // GOROOT,go的安装目录。
	D:\goprojects\demo01\src\treefds (from $GOPATH)   // 项目的gopath路径
	C:\Users\Administrator\go\src\treefds   // 全局gopath路径
```

5.**语法介绍：**
```go
fmt.Printf("%T %V\n", r, r)  // %T ->Type, %V -> Value. 可以打印出对象的类型与值。
```
- 一个目录下，只能有一个包。如果已经定义为main包，那么就不能定义其他的包了。
- go没有继承与多态，因此只有struct，没class.
- go中的Reader与Writer传的参数可以为文件，网络，slice
- ```闭包：```
	- 匿名函数同样被称之为闭包（函数式语言的术语）：它们被允许调用定义在其它环境下的变量。闭包可使得某个函数捕捉到一些外部状态，例如：函数被创建时的状态。另一种表示方式为：一个闭包继承了函数所声明时的作用域。这种状态（作用域内的变量）都被共享到闭包的环境中，因此这些变量可以在闭包中被操作，直到被销毁。闭包经常被用作包装函数：它们会预先定义好 1 个或多个参数以用于包装。另一个不错的应用就是使用闭包来完成更加简洁的错误检查。
	- 所谓闭包是指内层函数引用了外层函数中的变量或称为引用了自由变量的函数，其返回值也是一个函数，了解过的语言中有闭包概念的像 js，python，golang 都类似这样。
	- **官方解释（译文）**：Go 函数可以是一个闭包。闭包是一个函数值，它引用了函数体之外的变量。 这个函数可以对这个引用的变量进行访问和赋值；换句话说这个函数被“绑定”在这个变量上。
- defer关键字：https://www.jianshu.com/p/5b0b36f398a2
- recover:仅在defer中调用，可以获取panic的值，如果无法处理，那么可以重新panic.
- 
1、http报文格式：
- 起始行(请求行，响应行)
- 首部
- 实体  

2、可以通过头部的属性Content-Type来区分不同的资源类型。 
 
访问www.baidu.com的logo的响应Headers
```java
Accept-Ranges: bytes　　// 可以分段下载图片
Cache-Control: max-age=315360000
Content-Length: 7877
Content-Type: image/png　// 类型为image
Date: Sun, 25 Nov 2018 07:41:46 GMT
Etag: "1ec5-502264e2ae4c0"  // 这个属性是是否利用cookie的东西,如果返回304就用。
Expires: Wed, 22 Nov 2028 07:41:46 GMT
Last-Modified: Wed, 03 Sep 2014 10:00:27 GMT
Server: Apache
```
3、get与post的区别：  
- GET参数拼接在url，不支持二进制，且URL长度有上限,```不安全```。
- POST参数在实体，支持文本，二进制.没有大小限制。 

4、cookie的过期时间：
> 注意：cache-control: max-age=3110400.以这个属性为准(一个时间范围)。expire时间是一个准确的定值，不同的机器之间会有误差。  

5、keep-alive的作用：
- 建立长连接，避免频繁的创建连接与关闭连接。
- TCP慢开始。(影响性能比较小)。可能因为建立TCP连接时，慢开始无法快速的来处理网络的流量而带来了一定的影响。  

6、HTTP响应首部参数如下:
- 通用首部:
    - Date:报文创建时间
    - Tranfer-Encoding:实体传输编码方式
    - Cache-control:缓存指示(比如过期的最大时间)
- 请求首部
    - host：请求服务器主机名和端口号（http1.1必传）,可以与资源路径拼接为一个完整的url.(否则并不知道访问的资源在哪个应用或者主机上。)
    - user-agent：请求客户端信息(比如用chrome请求的)
    - accept-encoding:客户端能接受的编码方式
- 响应首部 
    - server：服务器端服务器、程序信息
    - accept-ranges:服务器可以接受的范围类型
- 实体首部(比如请求实体(POST)和响应实体)
    - content-type：实体类型 application/json
    - content-length:实体长度
    - content-encoding：实体编码方式 gzip
    - content-range:服务器可以接受的范围类型  

7、curl参数详解：https://www.cnblogs.com/duhuo/p/5695256.html
- i 可以看到请求的详细信息。
- v --trace： 交互详细信息 
- X：请求方法 GET POST
- d：post请求实体
- -compressed：解压response

- b：请求时使用指定cookie文件
- c：保存返回的cookie到指定文件

- x：指定代理服务器地址和端口

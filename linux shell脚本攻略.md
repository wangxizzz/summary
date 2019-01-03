### curl命令介绍：
1.curl --referer Referer_URL target_URL : 参照页(referer)是位于HTTP头的一个字段，用来标识用户是从哪个页面调到当前页面的。

2.curl http://www.baidu.com --cookie-jar baidu.cookie 把访问百度的cookie记录到本地文件。

3.curl -H "Accept-language: en" http://www.baidu.com -H "Host: www.baidu.com" --user-agent "Mozilla/5.0" -H 表示增加Header, 可以增加多个Header. --user-agent表示用户代理。

4.curl -I http://www.baidu.com  -I 参数或者--head表示只打印HTTP头部信息。

### 网络管理:
1.网关:一个网络中的设备如果想同另一个网络中的设备进行通信,就需要借助某个同时连接了两个网络的设备。这个特殊的设备被称为网关,它的作用是在不同的网络中转发分组.

2.route -n 可以查看

3.**ping** : Ping发送一个ICMP,回声请求消息给目的地并报告是否收到所希望的ICMP echo （ICMP回声应答）。检查网络是否通畅或者网络连接速度的命令.
- ping www.baidu.com -c 2 :  ping 命令发送了2个 echo 分组后就停止发送
- ping发送数据使用ICMP,ICMP协议通过IP协议发送的，IP协议是一种无连接的，不可靠的数据包协议.而TCP协议提供了可靠的需求.

4.
### 磁盘管理

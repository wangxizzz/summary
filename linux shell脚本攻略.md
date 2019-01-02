### curl命令介绍：
1.curl --referer Referer_URL target_URL : 参照页(referer)是位于HTTP头的一个字段，用来标识用户是从哪个页面调到当前页面的。

2.curl http://www.baidu.com --cookie-jar baidu.cookie 把访问百度的cookie记录到本地文件。

3.curl -H "Accept-language: en" http://www.baidu.com -H "Host: www.baidu.com" --user-agent "Mozilla/5.0" -H 表示增加Header, 可以增加多个Header. --user-agent表示用户代理。

4.curl -I http://www.baidu.com  -I 参数或者--head表示只打印HTTP头部信息。

### 磁盘管理

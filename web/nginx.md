1.正向代理与反向代理:

https://www.jianshu.com/p/208c02c9dd1d

2.ng的大致构成:
- 一个master,多个worker.master主要做加载配置,监控和重启worker.
- worker具体处理client的请求.
- worker是异步非阻塞,实现高效性:worker接收client端的请求,根据配置会转发到指定的域名服务器下面,如果数据没有返回,worker会把这个请求挂起,会继续去响应client端的请求,等刚刚那个请求的数据返回了,worker再把数据打回client.这就是异步非阻塞(AIO).

3.worker的主要模块:
- handlers: 处理客户端请求
    - ngx_http_rewrite_module , ngx_http_log_module , ngx_http_static_modul
e
- Filters:对请求进来做一些过滤,比如rewrite
    - ngx_http_header_filter_module
- Upstreams:ng的请求经过了Filter模块就会到达Upstreams模块,这个模块会把请求转发到后台.
    - ngx_h?p_proxy_module , ngx_h?p_fastcgi_module
- load balancers :请求要转到哪台机器.

4.ng的命令:
- /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf -t    
    - -c 指定配置文件
    - -t 检查配置文件
- ./nginx -t  检查配置文件
- ./nginx -s reload : 重启Nginx服务
- 

5.

## 敏捷开发:

1.敏捷提倡跨职能团队.

2.
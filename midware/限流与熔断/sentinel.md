# 启动相关
```
到  https://github.com/alibaba/Sentinel/releases  下载jar
启动命令：java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.0.jar

访问 localhost:8080  输入 账户密码：sentinel/sentinel 进入界面

应用程序想连接sentinel控制台：
配置 JVM 启动参数：  
-Dproject.name=sentinel-demo -Dcsp.sentinel.dashboard.server=127.0.0.1:8080 -Dcsp.sentinel.api.port=8719

```

# 流量控制：
https://www.fangzhipeng.com/springcloud/2019/08/20/ratelimit-guava-sentinel.html


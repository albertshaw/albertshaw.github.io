title: Nginx反向代理 Nodejs PM2
date: 2014-06-23 20:10:18
tags: [nginx, nodejs, pm2, 反向代理]
---
之前博客用的是基于nodejs的ghost，虽然功能还很少，但是做个博客够了，而且部署确实方便。
运行服务用到了pm2，反向代理用的nginx。
<!-- more -->
## PM2运行Nodejs服务
pm2官方给自己的定义是：
>pm2 is a process manager for Node apps with a built-in load balancer.

负载均衡我还没用到，想想目前也不是必须的，以后再说。简单的用的话，直接pm2 start一个nodejs的server.js文件，不过当然还是建议自己定义的一个`<pm2 server list>.json`文件。启动参数都定义好，还能一次启动多个服务，以后启动重启都方便多了。`pm2 start <pm2 server list>.json`, `pm2 stop <pm2 server list>.json`, `pm2 restart <pm2 server list>.json`。json文件栗子如下:

```json
[{
  "name"        : "my-ghost",
  "script"      : "Ghost/index.js",
  "cwd"         : "/var/www/blog",
  "error_file" : "ghost-err.log",
  "out_file"   : "ghost-out.log",
  "pid_file"   : "ghost.pid",
  "one_launch_only"  : "true",
  "env": {
      "NODE_ENV": "production"
  }
}]
```

##Nginx反向代理
我的VPS系统是Centos6
```
yum install nginx
```
安装完毕后`nginx -t`，这命令是测试配置文件是否正确的，顺带用来找找配置文件在哪了。
加入nodejs服务反向代理，挺方便的，就是这句`proxy_pass http://localhost:2368;`
```
server {
    listen 80;
    server_name blog.xiaol.me xiaol.me;

    location / {
        proxy_pass http://localhost:2368;
    }
}
```
然后`nginx`启动。

不知道是不是我用得不对，遇到一个很奇怪的点。因为之前还没打算用反向代理，直接nodejs服务监听80端口的， 这样当pm2 stop nodejs服务后，发现端口依然占用，导致nginx没启动起来。最后没辙还得`lsof -i tcp:80` `kill pid`这样。以后有空再看看`pm2 stop`后端口依然占用的问题吧。

> 08/14/2014：`pm2 stop`仍占用端口，应该用`pm2 kill`

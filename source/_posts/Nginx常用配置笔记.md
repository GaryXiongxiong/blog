---
title: Nginx常用配置笔记
date: 2020-09-02 10:23:07
tags:
- Nginx
categories:
- blog
---

## 正向代理与反向代理

正向代理是指客户端通过代理服务器作为跳板，访问不方便直接访问的服务器。就像我们希望购买海外的商品但不方便直接出国或联系国外的商家，我们就可以寻找代购，帮我们带着我们的钱找到海外商家，并带着我们想买的商品回来。在这个过程中，我们需要明确我们想买的商品是什么在哪里，要找的代购是谁。

![forward-proxy](/images/forward-proxy.png)

反向代理是指反向代理服务器代理原始服务器接收客户端请求并分发到原始服务器。类比到某些贴牌品牌。消费者带着购物需求找到该品牌，品牌商根据该需求从实际的生产厂家拿到商品并返回给消费者。消费者在这个过程中只知道品牌商是谁，并不知道实际的生产厂家是谁在那里，也不知道需求在厂家与商家之间是如何流转的。

![reverse-proxy](/images/reverse-proxy.png)

Nginx就是这样一个反向代理服务，有着内存占用小，并发能力强的特点。

## Nginx配置文件

Nginx主配置文件`/etc/nginx/nginx.conf`

```json
### main块开始，此块为全局配置块，会影响其他所有设置 ###

# user: 指定Nginx worker 进程运行的用户与用户组
user  nginx;
# worker_processes: 指定worker进程的数量
worker_processes  1;

# error_log: 定义全局错误日志文件地址与日志级别
error_log  /var/log/nginx/error.log warn;
# pid: 定义进程pid文件的存储位置
pid        /var/run/nginx.pid;

### main块结束 ###

### events块开始 ###
events {
    # worker_connections: Nginx每个进程的最大连接数
    worker_connections  1024;
    # BTW, Nginx 能抗的最大并发 = worker_processes * worker_connections / (2(静态)|4(动态))
}
### events块结束 ###

### http块开始 ###
http {
    # 文档类型定义文件位置
    include       /etc/nginx/mime.types;
    # 当文档类型不在定义中时默认的加载类型
    default_type  application/octet-stream;

    # log_format: 定义日志记录格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    # access_log: 输出日志位置与格式
    access_log  /var/log/nginx/access.log  main;

    # sendfile参数用于开启高效文件传输模式。将tcp_nopush和tcp_nodelay两个指令设置为on用于防止网络阻塞；
    # 这里还是有点迷惑
    sendfile        on;
    #tcp_nopush     on;

    # keepalive_timeout: 连接保持时常
    keepalive_timeout  65;

    #gzip  on;
	
    # 引入/etc/nginx/conf.d/下的所有配置文件
    include /etc/nginx/conf.d/*.conf;
}
```

默认配置文件 `etc/nginx/conf.d/default.conf`

```json
### server块开始，用于指定主机与端口，继承main块设定 ###
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

	### location块开始，配置网页位置，继承server块设置 ###
	# 定义代理地址: /
	# 此处可使用正则
    location / {
        # 定义要代理到的位置
        root   /usr/share/nginx/html;
        # 定义index名称
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
	

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #
    }
```

## 反向代理配置

```json
	location /index {
        proxy_pass http://jiangyixiong.top:8080/;
    }

	location ^~ /baidu/ {
        proxy_pass https://baidu.com;
    }

    location / {
        proxy_pass http://jiangyixiong.top;
    }
```

## 负载均衡配置

配置upstream(配置在server块之外)

```json
upstream my_server{
    server jiangyixiong.top:8080;
    server jiangyixiong.top:8081;
    server jiangyixiong.top:8082;
}
```

配置反向代理到该upstream

```json
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

	location / {
        proxy_pass http://my_server/;	
    }
}
```

以上配置为轮询，其余还有权重与ip哈希算法。

权重轮询:

```json
upstream my_server{
    server jiangyixiong.top:8080 weight=2;
    server jiangyixiong.top:8081 weight=5;
    server jiangyixiong.top:8082 weight=3;
}
```

## 动静分离配置

静态资源需放置于Nginx所在服务器

```json
location / {
    # 静态资源所在目录
    root /usr/share/nginx/html;
    # index文件
    index index.html;
}
```

可通过`autoindex`显示静态资源目录

```json
location / {
    root /usr/share/nginx/data;
    autoindex on;
    #代表展示静态资源的全部内容，以列表的形式展开
}
```
![auto-index](/images/auto-index.png)

## References:
>
> - [《Nginx学习笔记 基于docker》南城.南城](https://blog.csdn.net/m0_49558851/article/details/107786372)
> - [《Nginx快速上手》](https://www.bilibili.com/video/BV1W54y1z7GM?p=14)
> - [《Nginx的配置文件详解》古陵云烟](https://blog.csdn.net/wangbin_0729/article/details/82109693)



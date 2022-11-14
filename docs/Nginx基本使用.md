---
title: Nginx基本使用
date: '2022-11-11'
categories:   # 分类
 - SpringCloud
tags:   # 标签
 - 负载均衡
 - 反向代理
 - nginx
publish: true   # 发布
---

<p style="color:#7f7f7f">nginx的安装及使用，反向代理、负载均衡、lua缓存、lua限流、静态化</p>

# Nginx基本使用

![nginx](http://img.sybutterfly.top/img/202211142109274.png)

本文基于docker安装nginx和centos7，出现位置问题的话，建议直接关键字百度，这里我们采用docker容器的形式安装，相关命令如下

注意：基于docker安装后，不能直接使用127.0.0.1代理跳转，这样访问的是容器的本地ip，请使用服务器的公网ip来代理

```bash
# 生成容器
docker run --name nginx -p 9001:80 -d nginx
# 将容器nginx.conf文件复制到宿主机
docker cp nginx:/etc/nginx/nginx.conf ./nginx.conf
# 将容器conf.d文件夹下内容复制到宿主机，下面这个文件夹是nginx.conf中默认include的，主要是server块的内容，在不修改nginx.conf的前提下，需要导入
docker cp nginx:/etc/nginx/conf.d ./conf.d
# 将容器中的html文件夹复制到宿主机
docker cp nginx:/usr/share/nginx/html ./

# 直接执行docker rm nginx或者以容器id方式关闭容器
# 找到nginx对应的容器id
docker ps -a
# 关闭该容器
docker stop nginx
# 删除该容器
docker rm nginx
 
# 删除正在运行的nginx容器
docker rm -f nginx

docker run \
-p 80:80 \
--name nginx \
-v /home/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /home/nginx/conf/conf.d:/etc/nginx/conf.d \
-v /home/nginx/log:/var/log/nginx \
-v /home/nginx/html:/usr/share/nginx/html \
-d nginx:latest
```

## 1、nginx.conf配置结构

![20210731155655187](http://img.sybutterfly.top/img/202211142109845.png)

### 1.1、默认配置

```text
# 设置worker进程的用户，指的linux中的用户，会涉及到nginx操作目录或文件的一些权限
user  nginx;
# worker进程工作数设置
worker_processes  auto;

#错误日志
error_log  /var/log/nginx/error.log notice;
# 进程pid
pid        /var/run/nginx.pid;

# 设置工作模式的设置
# 默认使用epoll
events {
	
# 每个worker允许连接的客户端最大连接数
    worker_connections  1024;
}


http {
	
	#include 引入外部配置，提高可读性，避免单个配置文件过大
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    
    #日志配置
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    
    # sendfile使用高效文件传输，提升传输性能。启用后才能使用tcp_nopush，是指当数据表累积一定大小后才发送，提高了效率。
    sendfile        on;
    #tcp_nopush     on;
    
    # 设置客户端与服务端请求的超时时间，保证客户端多次请求的时候不会重复建立新的连接，节约资源损耗。
    keepalive_timeout  65;
    
    # 开启gizp压缩,html、js、css传送更快
    #gzip  on;
    
    # 导入该目录下的文件，主要是server的配置
    include /etc/nginx/conf.d/*.conf;
}
```

### 1.2、server的配置

http区是我们的重点关注对象，基本上咱们的操作都是跟他相关的

server的默认配置，也就是`include /etc/nginx/conf.d/*.conf;`中导入的默认配置如下

```text
server {
    listen       80; #监听80端口
    listen  [::]:80; #监听IPV6的80端口
    server_name  localhost;  # 服务的名字

    location / { # location是路径配置，这里表示127.0.0.1/*下的所有路径都会被代理
        root   /usr/share/nginx/html; #目录配置
        index  index.html index.htm; # 未指定资源路径时，比如127.0.0.1:80时默认访问页面
    }

# 错误页面配置
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

}
```

#### 1.2.1、location.root的配置

```
location /img/ {
	alias /var/www/image/;
}
#若按照上述配置的话，则访问/img/目录里面的文件时，ningx会自动去/var/www/image/目录找文件
location /img/ {
	root /var/www/image;
}
#若按照这种配置的话，则访问/img/目录下的文件时，nginx会去/var/www/image/img/目录下找文件
```



## 2、反向代理

### 2.1、准备工作

这里你需要准备一个基本的springboot的jar包，或者tomcat包（linux下tomcat的安装这里不说了）

基本结构只需要一个controller和跳转页面即可，端口配置8080

![image-20221111201025222](http://img.sybutterfly.top/img/202211142109068.png)

打包后发送到服务器，后台运行jar包命令如下

```bash
java -jar xxxx.jar &   #后台启动
ps -ef | grep java   #查看java进程
kill -9 pid  #杀死进程
```

### 2.2、修改nginx.conf文件

```text
    # include /etc/nginx/conf.d/*.conf;   这里不需要这个了，当然你也可以用下面的部分，覆盖该文件
    
    server {
    listen       80;
    server_name  127.0.0.1;

    location /test {  #表示host:ip/test的请求转发到下面host:port/test进行处理
	proxy_pass http://host:port;  #这里的端口改成自个儿springboot项目的端口,非docker安装的nginx的host可以直接用127.0.0.1，docker请用公网ip
	# 下面的不是必选项
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
    }
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
```

### 2.3、测试

>重启nginx容器

```bash
docker restart nginx
```

![image-20221111204731213](http://img.sybutterfly.top/img/202211142109290.png)

访问，出现你springboot项目中的预期页面表示成功

## 3、负载均衡

将2.中的项目复制2份，需要不同的端口，不同的响应页面，以便看出负载均衡的效果

这里我出现了3个实例，有一个一直无法访问的问题，具体原因不明

> 注意，请提前在防火墙放行端口哦~

### 3.1、修改nginx.conf

```text
    upstream myServerSetting { # 名称随意
        server 172.158.32.118:8080;
        server 172.158.32.118:8081;
    }
    server {
        listen       80;
        server_name  192.168.32.128;
        #charset koi8-r;
 
        #access_log  logs/host.access.log  main;
        location / {
            proxy_pass http://myServerSetting;
            root   html;
            index  index.html index.htm;
        }
     }
```

### 3.2、测试

同反向代理，你将看见不同的页面依次交替出现，因为nginx默认的负载均衡策略是轮询，这里就不给图片了，可以自行脑补哦

### 3.3、负载均衡策略

好像不止四种，这里只说四种了哈

#### 3.3.1、轮询（默认）

轮询方式是 Nginx 负载均衡默认的方式。顾名思义，所有请求都按照时间顺序分配到不同的服务上。如果某个服务器 Down 掉，可以自动将该服务器剔除，不再分发请求到该服务器上。

```text
    upstream myServerSetting { # 名称随意
        server 172.158.32.118:8080;
        server 172.158.32.118:8081;
     }
```

#### 3.2.2、权重（weight）

可以用 weight 关键字来指定每个服务器的权重比例，默认权重为 1，weight和访问比率成正比，权重越高被分发到的请求越多。通常用于后端服务机器性能不统一，将性能好的分配权重高来发挥服务器最大性能。

```text
    upstream myServerSetting { # 名称随意
        server 172.158.32.118:8080 weight=1;
        server 172.158.32.118:8081 weight=1;
     }
```

#### 3.3.3、ip_hash（一个客户端访问固定服务器）

每个请求都根据访问 ip 的 hash 结果分配，经过这样的处理，每个访客固定访问一个后端服务，可以解决 session 的问题。

如下配置（ip_hash 可以和 weight 配合使用）。

```text
upstream  myServerSetting {
       ip_hash; 
        server 172.158.32.118:8081 weight=1;
        server 172.158.32.118:8082 weight=2;
}
```

#### 3.3.4、fair（服务器响应快的优先分配）

按后端服务器的响应时间来分配请求，响应时间短的优先分配。

```text
upstream  myServerSetting {
       server    localhost:10001 weight=1;
       server    localhost:10002 weight=2;
       fair;  
}
```



4、Lua数据缓存

5、Lua限流

6、静态化（动静分离）
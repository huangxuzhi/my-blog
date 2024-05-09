+++
title = 'Nginx快速入门'
date = 2024-05-09T11:33:35+08:00
draft = false
+++

Nginx（engine x），一款高性能的开源Web服务器，并发能力高，在较低配置的服务器上也能轻松处理大量请求，请求响应时间低，平均响应时间可以控制在几毫秒之内，主要充当反向代理服务器、负载均衡器、HTTP缓存和虚拟主机。

## 常用命令
```nginx
#启动nginx（不加-c参数使用默认的配置文件）
nginx -c 指定配置文件

# 重新加载配置文件
nginx -s reload

# 检查配置文件语法是否正确
nginx -t -c 指定配置文件

# 快速关闭nginx
nginx -s stop

# 等工作进程处理完后关闭nginx
nginx -s quit
```

## 配置文件结构
```nginx
worker_processes  1;


events {
    # 指定了每个 Nginx 工作进程能够同时处理的最大客户端连接数，即每个工作进程能够同时建立的TCP 连接数的上限。
    worker_connections  1024;
}

# http块用于配置HTTP服务器的行为，包括HTTP请求、反向代理、负载均衡等。http块通常包含一个或多个server块。
http {

    # upstream可以定义一组服务器，用于负载均衡
    upstream server_list {
        ...
    }

    # server块用于配置一个虚拟主机（或称为服务）。每个server块定义了一个服务器监听的端口、主机名、网站根目录等。一个Nginx实例可以有多个server块。
    server {

        # location块用于配置请求匹配的URL位置，可以指定不同的处理方式。例如，静态文件的服务、反向代理的路径等。
        location / {
        }

        ...更多location
    }
}
```
## location匹配规则
匹配location块可以有多种方式，写法：`location [空格 | = | ~ | ~* | ^~ | @] /uri/`，不同的符号方式不同。
- `空格`, 就是不写符号，匹配以指定patern开头的url。
- `=`，表示精准匹配，如果找到，立即停止搜索并立即处理此请求。
- `~`，表示执行一个正则匹配，区分大小写匹配。
- `~*`，表示执行一个正则匹配，不区分大小写匹配。
- `^~`，跟空格类似，前缀匹配，但是优先级比空格高。
- `@`，用于定义一个 Location块，且该块不能被外部Client 所访问，只能被Nginx内部配置指令所访问，比如try_files 或 error_page。

Windows系统使用nginx的时候，`~`和`~*`都不区分大小写。因为windows的文件系统不区分大小写，比如A.txt和a.txt会视为同一个文件。

匹配顺序：

1、首先找`=`的匹配规则，如果找到了，则选择这个location。

2、如果第一步没找到，则找字符串匹配，包括`空格`和`^~`，找到最长的字符串匹配，如果这个最长的字符串匹配是`^~`匹配，则选择这个location。

3、如果第二步没找到，或者最长的字符串匹配是`空格`匹配，则继续找正则匹配，包括`~`和`~*`，找最上面的匹配。

4、如果第三步没找到，则选择第二步找到的location。

### 示例
配置文件
```nginx
location = / {
    [配置1]
}

location / {
    [配置2]
}

location /documents/ {
    [配置3]
}

location ^~ /images/ {
    [配置4]
}

location ~* \.(gif|jpg|jpeg)$ {
    [配置5]
}
```
使用上述配置，"/"请求会匹配配置1，"/index.html"会匹配配置2，"/documents/document.html"会配匹配配置3，"/images/1.gif"会匹配配置4，"/documents/1.jpg"会匹配配置5。

## 主要使用场景

### 1. 静态文件服务

Nginx可以快速高效地提供静态文件服务，比如HTML、CSS、JavaScript和图像文件。只需简单的配置，Nginx就可以直接提供静态文件，不用借助复杂的应用服务器。

示例
```nginx
server {
    listen 80;
    # example.com是域名
    server_name example.com;

    root /var/www/html; # 静态文件存放目录

    location / {
        # 这个配置会尝试直接返回请求的文件，如果文件不存在，则返回 404 错误
        try_files $uri $uri/ =404;
    }
}
```
使用`expires xx`来启用缓存

`expires xx`用于设置浏览器缓存的过期时间。它告诉浏览器在多长时间内可以使用缓存的资源而不必重新请求服务器获取新的内容。该指令会控制`Expires`和`Cache-Control`两个请求头，告诉浏览器资源过期的时间，从而使浏览器下次请求时使用缓存，减少流量消耗。
```nginx
location / {
    root   html;
    index  index.html index.htm;
    try_files  $uri $uri/ /index.html;
    expires 10d;
}
```


### 2. 反向代理

什么是反向代理

正向代理跟反向代理的区别，通俗来说，正向代理代理客户端，比如客户端A原来访问服务器C，现在A使用了正向代理，先请求代理服务器B，由b再访问服务器C。

反向代理代理服务器，比如客户端A原来访问服务器C，现在C使用了反向代理，A变成先请求代理服务器B，由B再访问服务器C。（此时Nginx扮演的是代理服务器B的角色）

作为反向代理，Nginx接收客户端的请求，并将其转发到后端的应用服务器。这种模式非常适用于负载均衡、SSL终端、缓存和安全过滤等需求。

示例
```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080; # 将请求转发到本地服务
    }
}
```
反向代理经常用到的其他指令：
- `proxy_set_header`，转发给后端服务前，添加请求头，比如
    - `proxy_set_header Host $host`，设置Host请求头
    - `proxy_set_header X-Real-IP $remote_addr`，设置X-Real-IP请求头
    - `$host`和`$remote_addr`是nginx内置的变量，具体含义可参考文末。
- `proxy_connect_timeout`，配置跟后端服务的连接超时时间。


### 3. 负载均衡

Nginx可以将客户端请求分发到多个后端服务器，以实现负载均衡。通过配置不同的负载均衡算法，比如轮询、IP哈希、最小连接等，可以根据需求对请求进行分发。

示例
```nginx
# 定义一组服务器
upstream backend_servers {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend_servers;
    }
}
```

负载均衡默认使用轮询算法，还有以下几种：

加权轮询，权重越高，转发的频率越高。
```nginx
upstream backend_servers {
    server 127.0.0.1:8080 weight=1;
    server 127.0.0.1:8081 weight=2;
}
```
<br></br>
ip_hash，它根据客户端的 IP 地址将请求分发到后端服务器，以确保同一客户端的所有请求都被发送到同一台服务器上。这在某些情况下可以确保会话一致性，特别是对于需要在会话之间共享状态的应用程序来说很有用。
```nginx
upstream backend_servers {
    ip_hash;
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}
```
<br></br>
最少连接数，它会将新的请求分配给当前连接数最少的后端服务器。这种算法的目标是尽可能地将负载均衡在各个服务器上，以确保所有服务器的负载尽可能接近。
```nginx
upstream backend_servers {
    least_conn;
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
}
```

### 4. 配置HTTPS
```nginx
server {
  listen 443 ssl http2 default_server;   # SSL 访问端口号为 443
  server_name xxx;         # 填写绑定证书的域名

  ssl_certificate 证书文件路径;          # pem格式
  ssl_certificate_key 私钥文件路径;      # key格式
  ssl_session_timeout 10m;

  ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
  
  location / {
    root         /usr/share/nginx/html;
    index        index.html index.htm;
  }
}
```

### 5. 开启gzip压缩
```nginx
http {
    ...
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    ...
}
```

还有一些重要的配置项控制压缩的行为和性能：

- `gzip_comp_level`: 指定压缩级别，范围是1到9，数字越大压缩效果越好，但压缩所需的CPU时间也会增加。默认值是6。
- `gzip_types`: 指定需要压缩的文件类型。默认情况下，Nginx只会对text/html类型的文件进行压缩。你可以添加其他类型，如text/css、application/javascript等。注意，尽量不要压缩图片等已经压缩过的文件，因为这样会浪费服务器资源。
- `gzip_min_length`: 指定最小文件大小，小于该值的文件不会被压缩。默认值是20字节。
- `gzip_buffers`: 指定压缩缓冲区大小，用于存储压缩后的数据。默认值是48k。你可以根据服务器的内存情况来调整这个值。
- `gzip_disable`: 指定禁用gzip压缩的条件。例如，你可以通过配置gzip_disable "MSIE [1-6]\.";来禁用对IE6及以下版本浏览器的压缩。
- `gzip_proxied`: 指定在什么情况下启用压缩。默认情况下，只有在upstream服务器返回的响应状态码为200、206、302、401或者404时才会启用压缩。

## 附：nginx内置变量
|  变量名   | 含义  |
|  ----  | ----  |
| $uri  | 请求的URI，不包括查询参数 |
| $request_uri  | 完整的请求URI，包括查询参数 |
| $args  | 查询参数 |
| $arg_PARAMETER  | GET请求中变量名PARAMETER参数的值 |
| $http_HEADER  | HTTP请求头中的内容，HEADER为HTTP请求中的内容转为小写，-变为_(破折号变为下划线)，例如：$http_user_agent(Uaer-Agent的值)，$http_host(请求头Host的值)、$http_referer(请求头Referer的值) |
| $remote_addr  | 客户端IP地址 |
| $remote_port  | 客户端的端口 |
| $server_addr  | 当前服务器的IP地址 |
| $server_port  | 当前服务器的端口号 |
| $request_method  | 请求方法，如GET、POST等 |
| $status  | 服务器响应的HTTP状态码 |
| $scheme  | 请求的协议，如http或https |
| $host  | 请求中的主机头(Host)字段，如果请求中的主机头不可用或者空，则为处理请求的server名称(处理请求的server的server_name指令的值)。值为小写，不包含端口。 |

## 参考
- [1] [Nginx 从入门到实践，万字详解！](https://juejin.cn/post/6844904144235413512#heading-25)
- [2] [Nginx 极简教程](https://github.com/dunwu/nginx-tutorial?tab=readme-ov-file)
- [3] [吐血整理-关于Nginx 的 location 匹配规则总结看这一篇就够了](https://juejin.cn/post/6908623305129852942)

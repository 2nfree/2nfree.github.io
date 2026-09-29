# Nginx-Proxy-Config


# 前言

- 正向代理：用户通过代理服务器直接访问 Internet，叫做正向代理。
- 反向代理：用户将请求发送给代理服务器，代理服务器根据请求将请求转发到 Internet 中，得到请求后再返回给用户。
- 负载均衡：通过反向代理服务器将请求分发到多个服务器中。

# Nginx 常用命令

```bash
#查看 nginx 版本号
nginx -v

#关闭 nginx
nginx -s stop

#启动 nginx
nginx

#重新加载 nginx（平滑重启）
nginx -s reload

#测试配置文件
nginx -t
```

# Nginx 配置文件

nginx 配置文件由三部分组成

## 第一部分：全局块

从配置文件开始到 events 块之间的内容，主要会设置一些影响 nginx 服务器整体运行配置指令

```nginx
#工作线程：
#此值越大，nginx处理并发的能力越强（会受到硬件的限制）
#一般设置为auto，会自动识别cpu核数配置
work_processes auto;

#执行用户
user www www;
```

## 第二部分：events 块

影响 nginx 服务器与用户网络的连接数

```nginx
events {
  #支持的最大连接数
  worker_connections  65535;
}
```

## 第三部分：http块（包括server块和全局块）

包括文件引入、MIME-TYPE定义、日志自定义、连接超时时间、单链接请求数上限等

```nginx
http {
  include            mime.types;
  default_type       application/octet-stream;
  sendfile           on;
  #长连接超时时间
  keepalive_timeout  65;
  server_tokens      off;
  #限制请客户端请求体的大小
  client_max_body_size 50m;
  #开启gzip压缩
  gzip on;
  gzip_min_length 8000;
  gzip_comp_level 3;
  gzip_buffers 4 8k;
  gzip_types text/plain text/css application/xml image/png image/gif image/jpeg image/jpg font/ttf font/otf image/svg+xml application/x-javascript;
  gzip_disable "MSIE [1-6]\.";
  #日志格式
  log_format json '{"@timestamp": "$time_iso8601",'
    '"server_ip": "$server_addr",'
    '"client_ip": "$remote_addr",'
    '"server_name": "$server_name:$server_port",'
    '"method": "$request_method",'
    '"request": "$request_uri",'
    '"url": "$uri",'
    '"query": "$args",'
    '"status": "$status",'
    '"user_agent": "$http_user_agent",'
    '"referer": "$http_referer",'
    '"response_time": "$upstream_response_time",'
    '"x_forwarded_for": "$http_x_forwarded_for",'
    '"send_bytes": "$bytes_sent"}';
    # 服务日志（该设置可以在server层为不同虚拟主机设置）
    # access_log /data/logs/nginx/access_nginx.log json;
    # error_log /data/logs/nginx/error_nginx.log error;
    server {
      .........
    }
}
```

## 第四部分：server块

设置不同代理服务配置

```nginx
server {
  listen 80;
  server_name [域名];
  location / {
    #工作根目录
    root html;
    #默认主页
    index.html index.htm;
  }
}
```

# Nginx反向代理

## 代理示例配置

```nginx
server {
  listen 80;
  server_name 127.0.0.1;
  
  location / {
    #配置反向代理
    proxy_pass http://127.0.0.1:8080;
    index index.html index.htm;
  }
}
```

```nginx
server {
  listen 80;
  server_name 127.0.0.1;
  
  location ~ /login/ {
    #为不同url配置不同的反向代理
    proxy_pass http://127.0.0.1:8080;
  }
  
  location ~ /regist/ {
    #为不同url配置不同的反向代理
    proxy_pass http://127.0.0.1:8081;
  }
}
```

## 代理配置解析

```nginx
#指定代理位置
proxy_pass [代理名称];

#设置代理使用的http版本
proxy_http_version 1.1;

#代理web socket设置
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;

#指定修改被代理服务器返回的响应头中的location头域跟refresh头域数值
#proxy_redirect    off;

#设置代理请求头
proxy_set_header   Host              $host;
proxy_set_header   X-Real-IP         $remote_addr;
proxy_set_header   X-Port 443;
proxy_set_header   X-Server-Name [域名];
proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header   X-Forwarded-Proto $scheme;

#配置重试机制，参数为在那些状态下重试
proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;

#http请求无法立即被处理，被放在nginx的待处理池中等待被处理。
#此参数为等待的最长时间，默认为60秒
proxy_connect_timeout      90

#http请求被服务器处理完后，把数据传返回给Nginx的用时，默认60秒
proxy_send_timeout         90;

#http请求被处理后，nginx会等待处理结果，也就是容器返回的response。
#此参数即为服务器响应时间，默认60秒
proxy_read_timeout         86400;

#设置缓冲区大小,从被代理服务器取得的响应内容,会先读取放置到这里
#响应header通常位于这部分响应内容里边。设置的过小，可能会产生502错误
proxy_buffer_size          4k;

#设置从被代理服务器读取应答内容的缓存区的数目和大小。
#数目可以任意定，但是一个缓存区的大小一般就是4k或者8k
proxy_buffers              4 32k;

#proxy_busy_buffers_size不是独⽴的空间，他是proxy_buffers和proxy_buffer_size的⼀部分。
#nginx会在没有完全读完后端响应的时候就开始向客户端传送数据，所以它会划出⼀部分缓冲区来专门向客户端传送数据(这部分的⼤⼩是由proxy_busy_buffers_size来控制的，建议为proxy_buffers中单个缓冲区⼤⼩的2倍)，然后它继续从后端取数据，缓冲区满了之后就写到磁盘的临时⽂件中。
proxy_busy_buffers_size    64k;

#设置代理缓存存放位置
proxy_temp_path /data/temp;

#设置临时文件的大小
#临时⽂件由proxy_max_temp_file_size和proxy_temp_file_write_size这两个指令决定。
#proxy_temp_file_write_size是⼀次访问能写⼊的临时⽂件的⼤⼩，默认是proxy_buffer_size和proxy_buffers中设置的缓冲区⼤⼩的2倍，Linux下⼀般是8k。
#proxy_max_temp_file_size指定当响应内容⼤于proxy_buffers指定的缓冲区时, 写⼊硬盘的临时⽂件的⼤⼩. 如果超过了这个值, Nginx将与Proxy服务器同步的传递内容, ⽽不再缓冲到硬盘. 设置为0时, 则直接关闭硬盘缓冲
proxy_max_temp_file_size   0;
proxy_temp_file_write_size 64k;
```

# 负载均衡

## 负载均衡配置示例

```nginx
upstream myserver{
  server [ip]:[port];
  server [ip]:[port];
}

server {
  listen 80;
  server_name 127.0.0.1;
  
  location / {
    #设置代理到upstream
    proxy_pass http://myserver;
    index index.html index.htm;
  }
}
```

## 负载均衡策略

### 轮询（默认）

按时间顺序逐一分配到不同的后端服务器，后端服务器 down 的时候，能自动剔除

```nginx
upstream myserver{
  server [ip]:[port];
  server [ip]:[port];
}

server {
  listen 80;
  server_name 127.0.0.1;
  
  location / {
    proxy_pass http://myserver;
    index index.html index.htm;
  }
}
```

### 权重

weight，权重越高，分配越多

```nginx
upstream myserver{
  server [ip]:[port] weight=5;
  server [ip]:[port] weight=10;
}

server {
  listen 80;
  server_name 127.0.0.1;
  
  location / {
    proxy_pass http://myserver;
    index index.html index.htm;
  }
}
```

### ip_hash

每个请求按访问 ip 的 hash 结果分配，这样每个访客固定访问一个后端服务器，可以解决 session 的问题

```nginx
upstream myserver{
  ip_hash;
  server [ip]:[port];
  server [ip]:[port];
}

server {
  listen 80;
  server_name 127.0.0.1;
  
  location / {
    proxy_pass http://myserver;
    index index.html index.htm;
  }
}
```

### fair（不是内建负载均衡使用的轮换的均衡算法）

```nginx
upstream myserver{
  server [ip]:[port];
  server [ip]:[port];
  fair;
}

server {
  listen 80;
  server_name 127.0.0.1;
  
  location / {
    root html;
    #配置反向代理的位置
    proxy_pass http://myserver;
    index index.html index.htm;
  }
}
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/nginx-proxy-config/  


# Nginx Compilation Installation and Configuration


# Prepare

 The current server is Centos7

```bash
# install dependencies
yum install -y gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel popt popt-devel

# Configure www users
groupadd www
useradd -g www www -s /sbin/nologin -M
mkdir -pv /home/www
chown -R www:www /home/www
```

# Install

## Download and Compile

```bash
#  Download source
mkdir /data/packages
cd /data/packages
wget http://nginx.org/download/nginx-1.18.0.tar.gz
tar -zvxf  nginx-1.18.0.tar.gz && cd nginx-1.18.0

#compile and install
./configure \
--prefix=/usr/local/nginx \
--user=www \
--group=www \
--with-file-aio \
--with-http_stub_status_module \
--with-http_gzip_static_module \
--with-http_flv_module \
--with-http_ssl_module \
--with-http_v2_module \
--with-stream \
--with-pcre
make && make install

# create log folder
mkdir -pv /data/logs/nginx && chown -R www.www /data/logs/nginx

# remove Default Configuration
mv -f /usr/local/nginx/conf/nginx.conf /usr/local/nginx/conf/nginx.conf.default

# create website configuration folder
mkdir -p /usr/local/nginx/conf/vhost

# create softlink
ln -sv /usr/local/nginx/sbin/nginx /usr/local/bin/nginx
```

## Create nginx.conf configuration file

```bash
cat > /usr/local/nginx/conf/nginx.conf <<"EOF"
# working as www
user www www;

worker_processes  auto;

events {
  worker_connections  65535;
}
http {
  include            mime.types;
  default_type       application/octet-stream;
  sendfile           on;
  keepalive_timeout  65;
  server_tokens      off;
  client_max_body_size 50m;
  gzip on;
  gzip_min_length 8000;
  gzip_comp_level 3;
  gzip_buffers 4 8k;
  gzip_types text/plain text/css application/xml image/png image/gif image/jpeg image/jpg font/ttf font/otf image/svg+xml application/x-javascript;
  gzip_disable "MSIE [1-6]\.";

  log_format json escape=json '{"@timestamp": "$time_iso8601",'
    '"client_ip": "$remote_addr",'
    '"server_ip": "$server_addr",'
    '"hostname": "$hostname",'
    '"upstream_addr": "$upstream_addr",'
    '"server_name": "$server_name:$server_port",'
    '"method": "$request_method",'
    '"request": "$request_uri",'
    '"url": "$uri",'
    '"query": "$args",'
    '"status": "$status",'
    '"upstream_status": "$upstream_status",'
    '"user_agent": "$http_user_agent",'
    '"referer": "$http_referer",'
    '"request_time": $request_time,'
    '"response_time": $upstream_response_time,'
    '"display_response_time": $upstream_response_time,'
    '"upstream_connect_time": $upstream_connect_time,'
    '"upstream_header_time": $upstream_header_time,'
    '"x_forwarded_for": "$http_x_forwarded_for",'
    '"cookie": "$http_cookie",'
    '"send_bytes": "$bytes_sent"}';

    # log file output location
    # access_log '/data/logs/nginx/access-nginx.log' json;
    # error_log '/data/logs/nginx/error-nginx.log' error;
    
  map $http_upgrade $connection_upgrade {  
    default upgrade;
    ''      close;
  }

# IP access blocking
  server {
    listen       80 default;
    server_name  _;

    location / {
      return 404;
    }
  }
  # site folder
  include vhost/*.conf;
}
EOF
```

## Create Nginx Systemd

```bash
cat > /usr/lib/systemd/system/nginx.service << EOF
[Unit]
Description= The Nginx Web Server
After=network.target remote-fs.target nss-lookup.target
[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
KillSignal=SIGKILL
PrivateTmp= true
[Install]
WantedBy=multi-user.target
EOF

# start nginx
systemctl start nginx 
systemctl enable nginx
systemctl status nginx
```

# Remaining

## Logrotate

```bash
cat > /etc/logrotate.d/nginx <<"EOF"
/data/logs/nginx/*.log {
    daily
    rotate 15
    compress
    nodelaycompress
    ifempty
    dateext
    missingok
    postrotate
        [ -e /usr/local/nginx/sbin/nginx ] && /usr/local/nginx/sbin/nginx -s reload &>/dev/null
    endscript
}
EOF
```

## Site Config Example

The related uppercase configurations wrapped in '[]' need to be filled in by yourself.

```nginx
upstream [PROXY_NAME]{
    ip_hash;
    server [ip]:[port];
    server [ip]:[port];
    server [ip]:[port];
}


server {
  listen 80;
  server_name [DOMAIN];
  client_max_body_size 1024m;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl;
  server_name [DOMAIN];
  client_max_body_size 1024m;
  add_header Strict-Transport-Security "max-age=31536000";
  
  ssl_certificate  [SSL_CERT];
  ssl_certificate_key  [SSL_CERT_KEY];
  ssl_ciphers "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4!3DES!ADH";
  ssl_protocols TLSv1.2;
  ssl_prefer_server_ciphers on;

  location / {
    proxy_pass http://[PROXY_NAME];
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    #proxy_redirect     off;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Port 443;
    proxy_set_header   X-Server-Name     $server_name;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
    proxy_max_temp_file_size   0;
    proxy_connect_timeout      90;
    proxy_send_timeout         90;
    proxy_read_timeout         86400;
    proxy_buffer_size          4k;
    proxy_buffers              4 32k;
    proxy_busy_buffers_size    64k;
    proxy_temp_file_write_size 64k;

  }
  location ~ .* {
    proxy_pass  http://[PROXY_NAME];
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Fonwarded-For $proxy_add_x_forwarded_for;
  }
}
```


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/install-nginx/  


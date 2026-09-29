# Nginx Proxy Config


# What is a Proxy

At its core, a proxy acts as an intermediary between two systems. In the context of networks, it's a server that handles traffic between clients (like your web browser) and other servers on the internet.

- Forward Proxy:
  - Client-side: A forward proxy sits in front of a group of clients within a network.
  - Function:
    - Manage Outgoing Traffic: Controls and filters internet access for clients.
    - Security: Provides a protective layer to shield clients from malicious content or websites.
    - Caching: Stores frequently accessed web content to reduce bandwidth usage and improve load times.
    - Anonymization: Helps users mask their IP addresses to browse the web privately.

- Reverse Proxy
  - Server-Side: A reverse proxy sits in front of one or more web servers.
  - Function:
    - Traffic Distribution: Distributes incoming requests across multiple servers (load balancing), optimizing performance and preventing overload.
    - Security: Acts as a shield for backend servers, masking their true identities from direct client contact.
    - Caching: Stores static content, reducing the load on backend servers.
    - Content Delivery: Can optimize content for different devices and locations.

# Nginx Common Commands

```bash
#get nginx version
nginx -v

#stop nginx
nginx -s stop

#start nginx
nginx

#reload nginx (smooth restart)
nginx -s reload

#check config format
nginx -t
```

# Nginx Config

The nginx config file consists of three parts

## Part 1: Global

From the beginning of the configuration file to the events block
The configuration here is designed for nginx servers running

```nginx
# work_processes
# The higher the value, the better nginx can handle concurrency, but there are hardware limitations.
# It is recommended to set it to auto, which will automatically recognise the current host's core count
work_processes auto;

# run as www，default is the nginx user
user www www;
```

## Part 2: Event

The configuration here refers to setting the number of connections between the Nginx server and the user's network.

```nginx
events {
  # The worker_connections directive in your Nginx configuration file determines the maximum number of simultaneous connections that a single Nginx worker process can handle at any given time.
  worker_connections  65535;
}
```

## Part 3: Http

Includes: file introduction, MIME-TYPE definition, log customisation, connection timeout, maximum number of requests for a single link, etc.

```nginx
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

    # access_log /data/logs/nginx/access_nginx.log json;
    # error_log /data/logs/nginx/error_nginx.log error;
    server {
      .........
    }
}
```

## Part 4: Server

Each server block represents a distinct website or application, allowing Nginx to serve different content based on client requests.

```nginx
server {
  listen 80;
  server_name [DOMAIN];
  location / {
    # working root directory
    root html;
    # default homepage
    index.html index.htm;
  }
}
```

# Nginx Reverse Proxy

## Config Example

```nginx
server {
  listen 80;
  server_name 127.0.0.1;
  
  location / {
    # reverse proxy
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
    # different reverse proxies for different urls
    proxy_pass http://127.0.0.1:8080;
  }
  
  location ~ /regist/ {
    # different reverse proxies for different urls
    proxy_pass http://127.0.0.1:8081;
  }
}
```

## Some Reverse Proxy Config

```nginx
# specify proxy url or name
proxy_pass [PROXY_URL];

# http version
proxy_http_version 1.1;

# web socket
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;

# is modify the Location header field in responses (like redirects)
#proxy_redirect    off;

# proxy header
proxy_set_header   Host              $host;
proxy_set_header   X-Real-IP         $remote_addr;
proxy_set_header   X-Port 443;
proxy_set_header   X-Server-Name [DOMAIN];
proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header   X-Forwarded-Proto $scheme;

# retries and failovers
proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;

# http requests are not processed immediately and are placed in the nginx pending pool waiting to be processed.
# this parameter is the maximum waiting time, the default is 60 seconds.
proxy_connect_timeout      90

# the amount of time it takes for a http request to be processed by the server and returned to Nginx, default 60 seconds.
proxy_send_timeout         90;

# after the http request is processed, nginx waits for the result of the process
# this parameter is the server response time, the default is 60 seconds.
proxy_read_timeout         86400;

# the buffer size, the response from the proxy server will be read and placed here first.
# the response header is usually located inside this part of the response content. Setting it too small may result in a 502 error
proxy_buffer_size          4k;

# the number and size of the buffer from which the answer content is read by the proxied server.
# the number can be arbitrary, but the size of a cache is usually 4k or 8k.
proxy_buffers              4 32k;

# `proxy_busy_buffers_size` is not a proprietary space, it is a part of `proxy_buffers` and `proxy_buffer_size`.
# nginx will start sending data to the client before it has finished reading the backend's response, so it will set aside a portion of the buffer dedicated to sending data to the client (the size of this portion is controlled by proxy_busy_buffers_size, which is recommended to be two times the size of the single buffer in proxy_buffers), and then it will continue to fetch the data from the backend, and then write it to a temporary file on the disk when the buffer is full.
proxy_busy_buffers_size    64k;

# the proxy cache storage location
proxy_temp_path /data/temp;

# the size of the temporary file
# temporary file size is determined by `proxy_max_temp_file_size` and `proxy_temp_file_write_size`.
# `proxy_temp_file_write_size` is the size of the temporary file that can be written to in a single access, by default it is twice the size of the buffer set in `proxy_buffer_size` and `proxy_buffers`, and under Linux it is usually 8k.
# `proxy_max_temp_file_size` specifies the size of the temporary file to be written to the hard drive when the response is larger than the buffer specified by `proxy_buffers`. If this value is exceeded, Nginx will synchronise the delivery of the content with the Proxy server instead of buffering it to the hard disk. Setting this value to 0 disables buffering to the hard disc.
proxy_max_temp_file_size   0;
proxy_temp_file_write_size 64k;
```

# Load Balance

## config example

```nginx
upstream myserver{
  server [ip]:[port];
  server [ip]:[port];
}

server {
  listen 80;
  server_name 127.0.0.1;
  
  location / {
    # set proxy name
    proxy_pass http://myserver;
    index index.html index.htm;
  }
}
```

## Load Balance Policy

### Round Robin (default)

Distributes requests sequentially across a list of servers. Simple and easy, but doesn't account for server capacity differences.

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

### Weight

Assigned different weights to influence the distribution of traffic.

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

### IP Hash

Maps a client's IP address to a specific server, ensuring consistency for some types of applications.

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

### Fair

Nginx keeps track of how busy each upstream server is, including factors like active connections and request processing time.
When a new request arrives, Nginx will try to select the server that currently has the lightest load.

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
    proxy_pass http://myserver;
    index index.html index.htm;
  }
}
```


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/nginx-proxy-config/  


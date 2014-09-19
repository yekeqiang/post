# Nginx 反向代理 websocket

标签（空格分隔）： nginx websocket

---

最近有一个需求，就是需要使用 nginx 反向代理 websocket，经过查找一番资料，目前已经测试通过，本文只做一个记录

> 注： 看官方文档说 Nginx 在 1.3 以后的版本才支持 websocket 反向代理，所以要想使用支持 websocket 的功能，必须升级到 1.3 以后的版本，因此我这边是下载的 Tengine 的最新版本测试的

 1. 下载 tengine 最近的源码

    ```
    wget http://tengine.taobao.org/download/tengine-2.0.3.tar.gz
    ```
 2. 安装基础的依赖包
 
    ``` 
    yum -y install pcre*
    yum -y install zlib*
    yum -y install openssl*
    ```
 3. 解压编译安装
    ```
    tar -zxvf tengine-2.0.3.tar.gz
    cd tengine-2.0.3
    ./configure --prefix=安装目录
    make
    sudo make install
    ```
 
nginx.conf 的配置如下：

```
user apps apps;
worker_processes  4; # 这个由于我是用的虚拟机，所以配置的 4 ，另外 tengine 可以自动根据CPU数目设置进程个数和绑定CPU亲缘性
# worker_processes auto
# worker_cpu_affinity auto

error_log  logs/error.log;

pid        logs/nginx.pid;

#Specifies the value for maximum file descriptors that can be opened by this process.
worker_rlimit_nofile 65535;

events {
    use epoll;
    worker_connections  65535;
}

# load modules compiled as Dynamic Shared Object (DSO)
#
#dso {
#    load ngx_http_fastcgi_module.so;
#    load ngx_http_rewrite_module.so;
#}

http {
    include       mime.types;
    default_type  application/octet-stream;


    server_names_hash_bucket_size 128;
    client_header_buffer_size 4k;
    large_client_header_buffers 4 32k;
    client_max_body_size 80m;

    sendfile on;
    tcp_nopush     on;

    client_body_timeout  5;
    client_header_timeout 5;
    keepalive_timeout  5;
    send_timeout       5;


    open_file_cache max=65535 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 1;

    tcp_nodelay on;

    fastcgi_connect_timeout 300;
    fastcgi_send_timeout 300;
    fastcgi_read_timeout 300;
    fastcgi_buffer_size 64k;
    fastcgi_buffers 4 64k;
    fastcgi_busy_buffers_size 128k;
    fastcgi_temp_file_write_size 128k;

    client_body_buffer_size  512k;
    proxy_connect_timeout    5;
    proxy_read_timeout       60;
    proxy_send_timeout       5;
    proxy_buffer_size        16k;
    proxy_buffers            4 64k;
    proxy_busy_buffers_size 128k;
    proxy_temp_file_write_size 128k;

    gzip on;
    gzip_min_length  1k;
    gzip_buffers     4 16k;
    gzip_http_version 1.0;
    gzip_comp_level 2;
    gzip_types       text/plain application/x-javascript text/css application/xml;
    gzip_vary on;
    proxy_temp_path   /dev/shm/temp;
    proxy_cache_path  /dev/shm/cache levels=2:2:2   keys_zone=cache_go:200m inactive=5d max_size=7g;


    log_format log_access  '$remote_addr - $remote_user [$time_local] "$request" "$request_time" "$upstream_response_time"'
              '$status $body_bytes_sent "$http_referer" '
              '"$http_user_agent" $http_x_forwarded_for $host $hostname' ;

    #websocket 需要加下这个
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    include /home/apps/tengine/conf/test.com; 


}
```

test.com 的配置文件内容：

```
upstream test.com {
   server 192.168.1.5:9000;
}

server {
    listen       80;
    server_name  test.com;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location  ^~  /websocket {
        proxy_pass http://test.com;

        proxy_redirect    off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

}
```

## 参考资料

 1. 主要是一个websocket 的框架的介绍：http://blog.controlgroup.com/2013/10/17/simple-websockets-example-play-2-2-0/  
 2. nginx 的关于 websocket 官方文档：http://nginx.org/en/docs/http/websocket.html
 3. 开发者的博客：http://blog.fens.me/nodejs-websocket-nginx/ 
 4. rfc2616 文档介绍 Upgrade 的章节：http://tools.ietf.org/html/rfc2616#section-14.42


---
title: "Docker 中搭建 EMQX Standalone 并使用 Nginx 进行反向代理"
pubDatetime: 2024-07-10T20:11:31+08:00
slug: docker-emqx-standalone-nginx
tags: ["Docker", "EMQX", "Nginx"]
description: "服务器使用的国内阿里云，系统：Debian12，直接使用 root 登录"
---


# 1. 环境

服务器使用的国内阿里云，系统：Debian12，直接使用 root 登录
由于懒得备案所以没有使用域名，直接使用 ZeroSSL 白嫖 IP SSL证书保存到如下位置：

> [!IMPORTANT]
> /etc/nginx/certificate/certificate.crt;
> /etc/nginx/certificate/private.key;

# 2. Docker

如果没有安装 Docker，使用如下命令安装Docker：

```bash
curl -fsSL https://get.docker.com -o install-docker.sh
```

由于特殊原因导致下载不成功可以直接打开网页右键另存为然后上传服务器，然后使用如下命令使文件可执行：

```bash
chmod +x install-docker.sh
```

由于我的服务器是阿里云的，所以直接使用阿里云的 Docker-CE 镜像安装：

```bash
sh install-docker.sh --mirror Aliyun
```

docker 安装完成后，创建一个静态桥接网络，让我能够指定 emqx 容器的 ip 地址：

```bash
docker network create --driver=bridge --subnet=172.20.0.0/16 --gateway=172.20.0.1 static
```

# 3. EMQX

使用如下命令安装，这里不对外暴露任何端口，因为要使用 Nginx 进行反向代理：

```bash
docker run -d \
  --name EMQX \
  --network=static \
  --hostname=emqx \
  --ip=172.20.0.100 \
  --restart=always \
  -v emqx-data:/opt/emqx/data \
  -v emqx-log:/opt/emqx/log \
  emqx/emqx
```

# 4. Nginx 

直接使用如下命令安装在服务器上，没必要装在 Docker 里：

```bash
apt install nginx libnginx-mod-stream
```

安装完成后，修改 `/etc/nginx/nginx.conf`，添加如下配置，反向代理 MQTT 与 MQTT SSL，由于 TLS 在 Nginx 中，所以无需反向代理 8883 的 emqx ssl 端口：

> [!WARNING]
> 以下配置需要去EMQX Dashboard - 集群控制 - 监听器中将 tcp 协议中的代理协议设置为 true
> ws 协议高级设置中添加自定义配置：
> proxy_address_header = "X-Forwarded-For"
> proxy_port_header = "X-Forwarded-Port"

```plaintext file="nginx.conf"
stream {

  log_format access '$remote_addr [$time_local] '
                 '$protocol $status $bytes_sent $bytes_received '
                 '$session_time "$upstream_addr"';

  upstream emqx {
    server 172.20.0.100:1883;
  }

  # 反向代理 MQTT 到 <ip>:1883
  server {
    listen 1883;
    listen [::]:1883;

    access_log /var/log/nginx/mqtt.access.log access;
    error_log /var/log/nginx/mqtt.error.log;
    
    proxy_pass emqx;
    
    proxy_protocol on;
    proxy_connect_timeout 10s;
    # Default keep-alive time is 10 minutes
    proxy_timeout 1800s;
    proxy_buffer_size 3M;
    tcp_nodelay on;
  }

  # 反向代理 MQTT 到 <ip>:8883
  server {

    listen 8883 ssl;
    listen [::]:8883 ssl;
    
    # 设置 SSL Session 缓存时间否则会报错 Session Taken over
    ssl_session_cache shared:MQTT_SSL:10m;
    ssl_session_timeout 10m;
    # SSL 证书
    ssl_certificate /etc/nginx/certificate/certificate.crt;
    ssl_certificate_key /etc/nginx/certificate/private.key;
    ssl_verify_depth 2;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    access_log /var/log/nginx/mqtt.access.log access;
    error_log /var/log/nginx/mqtt.error.log;
    
    proxy_pass emqx;
    
    proxy_protocol on;
    proxy_connect_timeout 10s;
    # Default keep-alive time is 10 minutes
    proxy_timeout 1800s;
    proxy_buffer_size 3M;
    tcp_nodelay on;
  }
}
```

在 `/etc/nginx/sites-available/` 中创建 `emqx.conf`，添加如下内容，对 EMQX 中的 Dashboard，WebSocket 进行反向代理，由于 TLS 在 Nginx 中，所以无需反向代理 8084 的 emqx websocket ssl 端口：

```plaintext file="emqx.conf"
# 代理 EMQX Dashboard 到 https://<ip>/dashboard
server {

  listen 18083 ssl http2;
  listen [::]:18083 ssl http2;

  # SSL 证书
  ssl_certificate /etc/nginx/certificate/certificate.crt;
  ssl_certificate_key /etc/nginx/certificate/private.key;
  include /etc/nginx/options-ssl-nginx.conf;
  ssl_dhparam /etc/nginx/ssl-dhparam.pem;

  access_log /var/log/nginx/dashboard.access.log;
  error_log /var/log/nginx/dashboard.error.log;

  location / {

    proxy_pass http://172.20.0.100:18083/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}

# 反向代理 EMQX WebSocket 到 <ip>:8083/mqtt
server {

  listen 8083;
  listen [::]:8083;

  # 由于没有使用使用域名是IP证书所以这里是服务器IP
  server_name <ip>;

  access_log /var/log/nginx/ws.access.log;
  error_log /var/log/nginx/ws.error.log;

  location /mqtt {

    proxy_pass http://172.20.0.100:8083;
    
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    
    proxy_buffering off;
    
    proxy_connect_timeout 10s;
    proxy_send_timeout 3600s;
    proxy_read_timeout 3600s;
    
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}

# 反向代理 EMQX WebSocket SSL 到 <ip>:8084/mqtt
server {

  listen 8084 ssl;
  listen [::]:8084 ssl;

  # 由于没有使用使用域名是IP证书所以这里是服务器IP
  server_name <ip>;

  access_log /var/log/nginx/wss.access.log;
  error_log /var/log/nginx/wss.error.log;

  # 设置 SSL Session 缓存时间否则会报错 Session Taken over
  ssl_session_cache shared:WSS_SSL:10m;
  ssl_session_timeout 10m;
  ssl_certificate /etc/nginx/certificate/certificate.crt;
  ssl_certificate_key /etc/nginx/certificate/private.key;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers HIGH:!aNULL:!MD5;

  location /mqtt {

    proxy_pass http://172.20.0.100:8083;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    proxy_buffering off;
  }
}
```

再创建软连接到 `sites-enabled`：

```bash
ln -s /etc/nginx/sites-available/emqx.conf /etc/nginx/sites-enabled/
```

验证配置：

```bash
nginx -t
```

没有错误的话重新加载配置：

```bash
nginx -s reload
```

# 5. 访问

后台：

> [!TIP]
> https://<ip>:18083

MQTT：

> [!TIP]
> <ip>:1883

MQTT SSL：

> [!TIP]
> <ip>:8883

WebSocket：

> [!TIP]
> ws://<ip>:8083/mqtt

WebSocket SSL：

> [!TIP]
> wss://<ip>:8084/mqtt

参考：
> [使用 NGINX 反向代理 EMQX 时获取客户端真实 IP](https://www.emqx.com/zh/blog/getting-the-clients-real-ip-when-using-the-nginx-reverse-proxy-emqx)
> [Load Balance EMQX Cluster with NGINX](https://docs.emqx.com/en/emqx/latest/deploy/cluster/lb-nginx.html)

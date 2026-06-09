---
title: "Nginx 代理配置完整指南"
date: 2026-06-09
draft: false
tags: ["Nginx", "代理", "反向代理", "高并发"]
categories: ["运维"]
---

## 前言

Nginx 是一个高性能的 HTTP 和反向代理服务器，在生产环境中广泛用于负载均衡、静态资源服务和 API 网关。本文详细介绍 Nginx 代理配置的最佳实践，包括基础代理配置、SSL 处理、高并发优化等关键内容。

<!--more-->

## 一、基础代理配置示例

### 单个域名代理配置

创建虚拟主机配置文件 `/etc/nginx/conf.d/abcd.cn.conf`：

```nginx
server {
    listen 80;
    server_name abcd.cn;

    # 主代理配置
    location / {
        # 后端服务地址
        proxy_pass https://abcd.com;

        # ==================== SSL 配置 ====================
        # 解决 SSL SNI 握手问题（必须配置）
        proxy_ssl_server_name on;
        proxy_ssl_protocols TLSv1.2 TLSv1.3;
        
        # SSL 证书验证（根据需求调整）
        # proxy_ssl_verify on;
        # proxy_ssl_verify_depth 2;
        # proxy_ssl_trusted_certificate /etc/nginx/ssl/ca.crt;

        # ==================== 请求头透传 ====================
        # 传递真实客户端信息
        proxy_set_header Host abcd.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        
        # 透传客户端 User-Agent 和 Referer
        proxy_set_header User-Agent $http_user_agent;
        proxy_set_header Referer $http_referer;

        # ==================== 跳转处理 ====================
        # 禁用自动跳转，保持原始地址
        proxy_redirect off;

        # ==================== 高并发优化 ====================
        # 使用 HTTP/1.1 协议
        proxy_http_version 1.1;
        
        # 禁用 Connection 头，启用连接复用
        proxy_set_header Connection "";
        
        # 后端故障转移策略
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        
        # 后端重试次数
        proxy_next_upstream_tries 2;

        # ==================== 日志配置 ====================
        access_log /var/log/nginx/abcd.com.log json;
    }
}
```

### HTTPS 代理配置

如果需要对外提供 HTTPS 服务，可以添加 SSL 配置：

```nginx
server {
    listen 443 ssl http2;
    server_name abcd.cn;

    # SSL 证书配置
    ssl_certificate /etc/nginx/ssl/abcd.cn.crt;
    ssl_certificate_key /etc/nginx/ssl/abcd.cn.key;

    # SSL 优化配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_stapling on;
    ssl_stapling_verify on;

    # HSTS 配置
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass https://abcd.com;
        proxy_ssl_server_name on;
        proxy_ssl_protocols TLSv1.2 TLSv1.3;
        
        # 保持其他配置与 HTTP 版本一致
        proxy_set_header Host abcd.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}

# HTTP 自动跳转到 HTTPS
server {
    listen 80;
    server_name abcd.cn;
    return 301 https://$server_name$request_uri;
}
```

## 二、nginx.conf 全局配置

编辑主配置文件 `/etc/nginx/nginx.conf`：

```nginx
# ==================== 进程配置 ====================
user  nginx;
worker_processes  auto;  # 自动检测 CPU 核心数
worker_rlimit_nofile  65535;  # 单个进程最大文件描述符

# ==================== 日志配置 ====================
error_log  /var/log/nginx/error.log  warn;
pid        /var/run/nginx.pid;

# ==================== 事件模块 ====================
events {
    use epoll;  # 使用 epoll 事件驱动（Linux 推荐）
    worker_connections  65535;  # 单个进程最大连接数
    multi_accept        on;  # 一次接收多个连接
    accept_mutex        off;  # 关闭互斥锁，提高并发
}

# ==================== HTTP 核心配置 ====================
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # ==================== 字符集配置 ====================
    charset utf-8;

    # ==================== JSON 格式日志 ====================
    log_format  json  '{
        "@timestamp":"$time_iso8601",
        "host":"$remote_addr",
        "server":"$host",
        "request":"$request",
        "status":"$status",
        "body_bytes_sent":"$body_bytes_sent",
        "referer":"$http_referer",
        "ua":"$http_user_agent",
        "request_time":"$request_time",
        "upstream_response_time":"$upstream_response_time",
        "upstream_connect_time":"$upstream_connect_time",
        "upstream_header_time":"$upstream_header_time"
    }';

    # ==================== 全局优化 ====================
    server_tokens       off;  # 隐藏 Nginx 版本号
    sendfile            on;  # 启用零拷贝
    tcp_nopush          on;  # 合并 TCP 包
    tcp_nodelay         on;  # 禁用 Nagle 算法
    keepalive_timeout   60;  # 连接保持时间
    keepalive_requests  10000;  # 单个连接最大请求数
    client_header_timeout 15;  # 请求头超时
    client_body_timeout   15;  # 请求体超时
    send_timeout          15;  # 发送超时
    reset_timedout_connection on;  # 重置超时连接

    # ==================== 客户端限制 ====================
    client_max_body_size    10M;  # 最大请求体大小
    client_header_buffer_size 4k;  # 请求头缓冲区
    large_client_header_buffers 4 8k;  # 大请求头缓冲区

    # ==================== 代理缓存配置 ====================
    proxy_cache_path  /tmp/nginx_cache 
        levels=1:2           # 目录层级
        keys_zone=cache:100m # 内存缓存大小
        max_size=500m        # 磁盘最大容量
        inactive=60m         # 过期时间
        use_temp_path=off;   # 禁用临时文件
    
    proxy_cache_key   "$host$request_uri";  # 缓存键

    # ==================== 代理全局优化 ====================
    proxy_connect_timeout 5;   # 连接后端超时
    proxy_read_timeout    30;  # 读取后端响应超时
    proxy_send_timeout    5;   # 发送数据到后端超时
    proxy_buffering       on;  # 启用响应缓冲
    proxy_buffer_size     16k;  # 单个缓冲区大小
    proxy_buffers         8 32k;  # 缓冲区数量和大小
    proxy_temp_file_write_size 64k;  # 临时文件写入大小

    # ==================== Gzip 压缩 ====================
    gzip on;
    gzip_min_length 100;  # 最小压缩长度
    gzip_types 
        application/json 
        application/javascript 
        text/plain 
        text/css 
        application/xml;
    gzip_comp_level 6;  # 压缩级别（1-9）

    # ==================== 包含虚拟主机配置 ====================
    include /etc/nginx/conf.d/*.conf;
}
```

## 三、关键配置参数详解

### 1. 进程与连接配置

| 参数 | 说明 |
|------|------|
| `worker_processes auto` | 根据 CPU 核心数自动设置进程数 |
| `worker_rlimit_nofile 65535` | 增大文件描述符限制 |
| `worker_connections 65535` | 单进程最大并发连接数 |

### 2. SSL 代理配置

```nginx
# 启用 SNI（Server Name Indication）
proxy_ssl_server_name on;

# 指定 TLS 协议版本
proxy_ssl_protocols TLSv1.2 TLSv1.3;

# 可选：证书验证
proxy_ssl_verify on;
proxy_ssl_trusted_certificate /path/to/ca.crt;
```

### 3. 请求头透传

```nginx
# 传递真实客户端 IP
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

# 传递协议信息
proxy_set_header X-Forwarded-Proto $scheme;

# 传递原始主机名
proxy_set_header Host $host;
```

### 4. 高并发优化

```nginx
# 使用 HTTP/1.1
proxy_http_version 1.1;

# 启用连接复用（关键）
proxy_set_header Connection "";

# 后端故障转移
proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
```

### 5. 日志格式说明

| 变量 | 说明 |
|------|------|
| `$request_time` | 总请求耗时（秒） |
| `$upstream_response_time` | 后端响应时间 |
| `$upstream_connect_time` | 连接后端耗时 |
| `$upstream_header_time` | 接收后端首包时间 |

## 四、负载均衡配置

如果后端有多个服务节点，可以配置负载均衡：

```nginx
# 定义上游服务器组
upstream backend {
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080 backup;  # 备用服务器
    
    # 负载均衡算法（默认轮询）
    # least_conn;  # 最少连接数
    # ip_hash;    # IP 哈希
    # fair;       # 响应时间优先
}

server {
    listen 80;
    server_name abcd.cn;

    location / {
        proxy_pass http://backend;
        # 其他配置...
    }
}
```

## 五、常用配置片段

### WebSocket 代理

```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
}
```

### 文件上传优化

```nginx
server {
    # 增大上传限制
    client_max_body_size 100M;
    
    location /upload {
        proxy_pass http://backend;
        proxy_buffering off;  # 禁用缓冲，直接传递
        proxy_request_buffering off;
    }
}
```

### 静态资源缓存

```nginx
location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
    add_header ETag "";
}
```

## 六、配置验证与测试

```bash
# 检查配置语法
nginx -t

# 平滑重启（不中断服务）
nginx -s reload

# 查看运行状态
systemctl status nginx

# 查看连接数
ss -tlnp | grep nginx

# 查看实时日志
tail -f /var/log/nginx/access.log
```

## 七、安全建议

1. **隐藏版本信息**：设置 `server_tokens off`
2. **限制请求大小**：合理设置 `client_max_body_size`
3. **启用 HTTPS**：使用 Let's Encrypt 免费证书
4. **配置防火墙**：只允许必要端口访问
5. **定期更新**：保持 Nginx 版本最新

## 总结

Nginx 代理配置需要根据业务场景灵活调整。核心要点：

1. **SSL 配置**：必须启用 `proxy_ssl_server_name` 支持 SNI
2. **请求头透传**：正确传递客户端真实信息
3. **高并发优化**：启用连接复用和故障转移
4. **日志监控**：使用 JSON 格式便于日志分析

合理的配置可以显著提升服务的稳定性和性能！

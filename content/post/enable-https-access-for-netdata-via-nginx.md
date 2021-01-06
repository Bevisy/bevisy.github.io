---
title: "通过nginx为netdata提供https访问"
date: 2020-01-21T15:56:21+08:00
draft: false
tags: 
    - Netdata
    - Nginx
categories:
---
# nginx 代理 netdata 并添加鉴权

## netdata 安装

```sh
#!/bin/bash

docker run -d --name=netdata \
  -p 30002:19999 \
  -v /etc/passwd:/host/etc/passwd:ro \
  -v /etc/group:/host/etc/group:ro \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -e DO_NOT_TRACK=1 \
  --cap-add SYS_PTRACE \
  --security-opt apparmor=unconfined \
  netdata/netdata

```

## nginx 安装

```sh
#!/usr/bin/env bash

docker run -d \
    --name nginx \
    -v $(pwd)/html:/usr/share/nginx/html \
    -v $(pwd)/conf.d:/etc/nginx/conf.d \
    -v $(pwd)/ssl:/etc/nginx/ssl \
    -v $(pwd)/passwords:/etc/nginx/passwords \
    -p 8080:80 \
    -p 8443:443 \
    -e TZ="Asia/Chongqing" \
    nginx
# 需要准备的文件
.
├── conf.d
│   └── default.conf
├── html
│   ├── 50x.html
│   └── index.html
├── nginx-start.sh
├── passwords
└── ssl
    ├── nginx.crt
    └── nginx.key

```

## nginx 配置

```sh
upstream netdata {
    server 192.168.124.2:19999;
    keepalive 64;
}

server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    location = /netdata {
        return 301 /netdata/;
    }

    location ~ /netdata/(?<ndpath>.*) {
        proxy_redirect off;
        proxy_set_header Host $host;

        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_pass_request_headers on;
        proxy_set_header Connection "keep-alive";
        proxy_store off;
        proxy_pass http://netdata/$ndpath$is_args$args;

        gzip on;
        gzip_proxied any;
        gzip_types *;
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
    #}
}


server {
    listen 443 ssl;
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;
    keepalive_timeout   60;
    server_name localhost;
    server_tokens off;

    fastcgi_param   HTTPS               on;
    fastcgi_param   HTTP_SCHEME         https;

    access_log      /var/log/nginx/access.log;
    error_log       /var/log/nginx/error.log;

    location / {
      root /usr/share/nginx/html/;
      index index.html index.htm;
    }

    location = /netdata {
        return 301 /netdata/;
    }

    location ~ /netdata/(?<ndpath>.*) {
        proxy_redirect off;
        proxy_set_header Host $host;

        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_pass_request_headers on;
        proxy_set_header Connection "keep-alive";
        proxy_store off;
        proxy_pass http://netdata/$ndpath$is_args$args;

        gzip on;
        gzip_proxied any;
        gzip_types *;
    }
}

```

访问nginx http://nginxurl:80/netdata/ 或者 https://nginxurl:443/netdata/，验证生效。

## 生成 ssl 证书和密钥

```sh
# 有效时间 7 天
openssl req -x509 -nodes -days 7 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt -subj "/C=CN/ST=SiChuan/L=Chengdu/O=Example Inc./OU=Web Security/CN=localhost"
```

生成的证书未经权威机构认证，但是不影响使用，只是浏览器会提示非安全的证书。

## nginx 配置https 服务

```markdown
server {
    listen 443 ssl;
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;
    keepalive_timeout   60;
    server_name localhost;
    server_tokens off;

    fastcgi_param   HTTPS               on;
    fastcgi_param   HTTP_SCHEME         https;

    access_log      /var/log/nginx/access.log;
    error_log       /var/log/nginx/error.log;

    location / {
      root /usr/share/nginx/html/;
      index index.html index.htm;
    }

    location = /netdata {
        return 301 /netdata/;
    }

    location ~ /netdata/(?<ndpath>.*) {
        proxy_redirect off;
        proxy_set_header Host $host;

        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_pass_request_headers on;
        proxy_set_header Connection "keep-alive";
        proxy_store off;
        proxy_pass http://netdata/$ndpath$is_args$args;

        gzip on;
        gzip_proxied any;
        gzip_types *;
    }
}
```



## nginx 添加鉴权

nginx 目前开启的是全局 ssl ，即通过 443 端口进入的请求均为 https，但是upstream 上的服务依旧是 http。

### 创建认证文件，开启 nginx 基础认证

```sh
# 提示输入密码和确认密码
printf "username:$(openssl passwd -apr1)" > /etc/nginx/passwords
```

### 在 server directive 中开启认证

```sh
server {
    listen 443 ssl;
    # ...
    auth_basic "Protected";
    auth_basic_user_file passwords;
    # ...
    access_log      /var/log/nginx/access.log;
    error_log       /var/log/nginx/error.log;
}
```

### HTTP 自动跳转HTTPS 安全配置

参考 [Nginx 服务器证书安装](https://cloud.tencent.com/document/product/400/35244)

```sh
server {
    listen 443;
    #填写绑定证书的域名
    server_name www.domain.com; 
    ssl on;
    #网站主页路径。此路径仅供参考，具体请您按照实际目录操作。
    root /var/www/www.domain.com; 
    index index.html index.htm;   
    #证书文件名称
    ssl_certificate  1_www.domain.com_bundle.crt; 
    #私钥文件名称
    ssl_certificate_key 2_www.domain.com.key; 
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    location / {
       index index.html index.htm;
    }
}
server {
    listen 80;
    #填写绑定证书的域名
    server_name www.domain.com; 
    #把http的域名请求转成https
    rewrite ^(.*)$ https://$host$1 permanent; 
}
```



## 使用权威机构签名证书

### 查看证书内容

```sh
openssl x509 -in nginx.crt -noout -text
```

### 查看key内容

```sh
openssl rsa -in nginx.key -noout -text
```

### 申请腾讯云免费SSL证书

[腾讯云免费ssl证书申请](https://cloud.tencent.com/developer/article/1522482)



## Reference

[Running Netdata behind Nginx](https://docs.netdata.cloud/docs/running-behind-nginx/)

[Enabling TLS support](https://docs.netdata.cloud/web/server/#enabling-tls-support)

[Nginx 服务器证书安装](https://cloud.tencent.com/document/product/400/35244)
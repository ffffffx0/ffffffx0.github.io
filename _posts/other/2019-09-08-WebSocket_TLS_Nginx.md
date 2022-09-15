---
title: V2ray+WebSocket+TLS+Nginx
tagline: ""
last_updated: null
category : template
layout: post
tags : [template, myblog]
---
### 环境 centos7

### 安装certbot
```
 yum -y install certbot
```
#### 生成key
注意需要开放端口443,80，由于vps没开放端口，导致没法生成key
```
certbot certonly --standalone -d xxyy.tk -m xxyy@gmail.com
```
生成的文件位于
```
/etc/letsencrypt/live/xxyy.tk/privkey.pem
/etc/letsencrypt/live/xxyy.tk/fullchain.pem
```
### 安装配置Nginx

```
yum -y install nginx
systemctl enable nginx
```
nginx配置文件
```
# cat /etc/nginx/conf.d/v2ray.conf 
server {
    listen       443 ssl;
    server_name  xxyy.tk;

    ssl_certificate    /etc/letsencrypt/live/xxyy.tk/fullchain.pem;
    ssl_certificate_key    /etc/letsencrypt/live/xxyy.tk/privkey.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    error_page 497  https://$host$request_uri;

location /ray {
    proxy_pass       http://127.0.0.1:10000;
    proxy_redirect             off;
    proxy_http_version         1.1;
    proxy_set_header Upgrade   $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host      $http_host;
    }
}

```
### 安装配置V2Ray


```
bash <(curl -L -s https://install.direct/go.sh)
systemctl enable v2ray
```
v2ray配置文件

```
# cat /etc/v2ray/config.json
{
  "log" : {   //日志配置
    "access": "/var/log/v2ray/access.log", //访问日志
    "error": "/var/log/v2ray/error.log",   //错误日志
    "loglevel":"debug" //日志等级
  },
  "inbound": {
    "port": 10000,
    "listen":"127.0.0.1",
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": uuid",//你的id
          "alterId": 64
        }
      ]
    },
    "streamSettings": {
      "network": "ws",
      "wsSettings": {
      "path": "/ray"
      }
    }
  },
  "outbound": {
    "protocol": "freedom",
    "settings": {}
  }
}

```

### 关闭防火墙

```
systemctl stop firewalld.service
vi /etc/selinux/config
SELINUX=disabled
setenforce 0
```

### 启动v2ray&nginx

```
systemctl start v2ray
systemctl status v2ray
systemctl start nginx
systemctl status nginx
```
客户端配置

### 域名到期了
freenom.com不支持xx宝，算了 继续白嫖吧
重新申请域名
xxx.cf

#### freenom.com与cf域名解析变更
所以也需要重新去cf配置cdn解析
首先配置cf上的cdn解析，解析完之后再上freenom.com修改dns解析为cf的

#### nginx 配置以及证书变更

生成证书,注意先关闭端口 比如80或者443冲突
```
 certbot certonly --standalone -d xxx.cf  -m abc@gmail.com
```

修改nginx配置
```
    server_name  xxx.cf;
    ssl_certificate    /etc/letsencrypt/live/xxx.cf/fullchain.pem;
    ssl_certificate_key    /etc/letsencrypt/live/xxx.cf/privkey.pem;
```






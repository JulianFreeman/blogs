---
title: Certbot 基础
date: 2025-11-22 17:08:00 -0400
tags:
  - cert
  - nginx
---
## 安装

本文使用 nginx 作反向代理，所以要安装 nginx 和 certbot 的 nginx 插件

```shell
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```

## 添加 DNS 记录

因为添加 DNS 记录后广播到全网需要几分钟时间，因此这里先创建记录然后再进行之后的操作。

在 Cloudflare 中添加一条指向服务器 IP 的 A 记录。

## 创建 nginx 配置文件

创建下面的文件并修改网址和端口号

```nginx {3,5} title="/etc/nginx/sites-available/my-site"
server {
    listen 80;
    server_name demo.example.com;
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

启用配置

```shell
sudo ln -s /etc/nginx/sites-available/my-site /etc/nginx/sites-enabled/
```

测试配置并重启

```shell
sudo nginx -t
sudo systemctl reload nginx
```

## 创建证书

创建证书并直接修改 nginx 配置，适合第一次给这个配置申请证书的情况

```shell /demo.example.com/
sudo certbot --nginx -d demo.example.com
```

只创建证书，不修改 nginx 配置，创建完成之后需手动修改配置文件添加证书路径

```shell /demo.example.com/
sudo certbot certonly --nginx -d demo.example.com
```

创建之后测试重启

```shell
sudo nginx -t
sudo systemctl reload nginx
```

## 查看证书

```shell
sudo certbot certificates
```

## 更新证书

更新 30 天之内到期的证书

```shell
sudo certbot renew --nginx
```

## 吊销证书

先切换到根用户，这样命令行可以自动补全证书路径

```shell
sudo su
certbot revoke --cert-path /etc/letsencrypt/live/yourdomain.com/fullchain.pem
```

删除证书（可选，一般在上一个命令里就已经删除了）

```shell
certbot delete --cert-name yourdomain.com
```

## 安装证书

如果使用了 `certonly` 模式申请了证书，但是 nginx 的配置文件并没有添加 https 支持，此时除了手动修改 nginx 配置文件之外，也可以直接安装证书

```shell
certbot install --nginx
```

运行后会要求选择要安装哪个证书，选择后 certbot 会自己寻找对应的配置文件并更新。

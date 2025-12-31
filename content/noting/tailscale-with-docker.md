---
title: 在 Docker 中使用 Tailscale
date: 2025-12-30 19:51:00 -0400
tags:
  - docker
  - tailscale
---
## 使用场景

有些服务不想暴露到公网上去，但是又希望能有 https 加密，就可以使用 Tailscale 。但是如果直接把 Tailscale 安装到主机上，它不方便代理多个服务，因此最好还是在 docker 中使用。

## 准备工作

1. 有一个 Tailscale 账号，在 DNS 页面下选择一个喜欢的 Tailnet DNS name，然后在页面底部开启 HTTPS Certificates 。
2. 在 Tailscale 的 Settings 页面下的 Key 下，创建一个 auth key ，创建时可以打开 Reusable 和 Ephemeral 。**保存下来这个 key ，它只显示一次** 。
3. 一台内网 Ubuntu Server，安装好 docker 和 docker compose 。

## 编写 compose

本例以 Timetagger 服务为例，原本的 Timetagger 服务是这样写：

```yaml
services:
  timetagger:
    image: ghcr.io/almarklein/timetagger
    container_name: timetagger
    restart: unless-stopped
    ports:
      - 127.0.0.1:80:80
    volumes:
      - ./_timetagger:/root/_timetagger
    environment:
      - TIMETAGGER_BIND=0.0.0.0:80
      - TIMETAGGER_DATADIR=/root/_timetagger
      - TIMETAGGER_LOG_LEVEL=info
      - TIMETAGGER_CREDENTIALS={YOUR_CREDENTIALS}
```

使用 Tailscale 代理网络之后，就 **不能** 写 `ports` 绑定了，会冲突。然后再添加如下内容：

```yaml {15-32} title="~/timetagger/compose.yaml"
services:
  timetagger:
    image: ghcr.io/almarklein/timetagger
    container_name: timetagger
    restart: unless-stopped
    #ports:
    #  - 127.0.0.1:80:80
    volumes:
      - ./_timetagger:/root/_timetagger
    environment:
      - TIMETAGGER_BIND=0.0.0.0:80
      - TIMETAGGER_DATADIR=/root/_timetagger
      - TIMETAGGER_LOG_LEVEL=info
      - TIMETAGGER_CREDENTIALS={YOUR_CREDENTIALS}
    network_mode: service:ts
    depends_on:
      - ts
  ts:
    image: tailscale/tailscale:latest
    hostname: timetagger
    environment:
      - TS_AUTHKEY={YOUR_AUTH_KEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_SERVE_CONFIG=/config/serve.json
    volumes:
      - ./ts-state:/var/lib/tailscale
      - ./config:/config
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    restart: unless-stopped
```

其中的 `hostname` 会反映到域名上。

然后手动创建 `config/serve.json` 文件，并写入如下内容：

```json {11} title="~/timetagger/config/serve.json"
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "${TS_CERT_DOMAIN}:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:80"
        }
      }
    }
  },
  "AllowFunnel": {
    "${TS_CERT_DOMAIN}:443": false
  }
}
```

这里唯一需要注意的就是这行 `Proxy` ，其中的端口要跟服务在 docker 内网中的端口一致。本例中也就是前面的 `TIMETAGGER_BIND=0.0.0.0:80` 。

## 运行服务

运行 `docker compose up -d` 启动服务，然后在安装了 Tailscale 客户端并登陆了相同账号的设备中访问 `https://timetagger.your-tailnet-name.ts.net` ，转一会就能正常访问了。
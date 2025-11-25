---
title: 搭建 Pairdrop 和中继服务
date: 2025-11-25 12:03:00 -0400
tags:
  - pairdrop
  - turn
---
## 无中继服务

[这](https://github.com/schlagmichdoch/PairDrop/blob/master/docs/host-your-own.md) 是官方的部署文档，如果不部署中继服务的话，搭建很简单，文档里提供了 compose

```yaml
services:
    pairdrop:
        image: "lscr.io/linuxserver/pairdrop:latest"
        container_name: pairdrop
        restart: unless-stopped
        environment:
            - PUID=1000 # UID to run the application as
            - PGID=1000 # GID to run the application as
            - WS_FALLBACK=false # Set to true to enable websocket fallback if the peer to peer WebRTC connection is not available to the client.
            - RATE_LIMIT=false # Set to true to limit clients to 1000 requests per 5 min.
            - RTC_CONFIG=false # Set to the path of a file that specifies the STUN/TURN servers.
            - DEBUG_MODE=false # Set to true to debug container and peer connections.
            - TZ=Etc/UTC # Time Zone
        ports:
            - "127.0.0.1:3000:3000" # Web UI
```

但是这样部署的服务只能发现局域网内的连接，无法加入公共房间。

所以下面部署带中继服务的，并说一下踩过的坑。

## 带中继服务

[官方文档](https://github.com/schlagmichdoch/PairDrop/blob/master/docs/host-your-own.md#coturn-and-pairdrop-via-docker-compose) 里给了详细的操作步骤，跟着操作基本没什么问题，只不过个别地方需要注意。

第一步，为域名申请证书，可以按照常规流程自动安装证书，不过 nginx 的配置文件要添加以下几行

```nginx
server {
    ...
    
    location / {
        ...
        proxy_connect_timeout 300;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        ...
    }
    
    listen 443 ssl http2;
    expires epoch;
    ...
}
...
```

[官方](https://github.com/schlagmichdoch/PairDrop/blob/master/docs/host-your-own.md#automatic-http-to-https-redirect) 这里有推荐的 http 跳转 https 配置，可以参考。

第二步，创建 `ssl` 目录，过。

第三步，将证书文件拷贝到 `ssl` 目录内，这里只需要拷贝 `fullchain.pem` 和 `privkey.pem` 。

第四步，调整 `ssl` 目录的所有者，过。

第五步，创建 `dhparams.pem` 文件，这里需要花点时间。

第六步，拷贝模板创建 `rtc_config.json` 文件，然后修改文件里的域名和用户名、密码。

第七步，拷贝模板创建 `turnserver.conf` 文件，除了同样修改域名和用户名密码之外，这里有几个地方需要改一下

```ini
# 这里端口数不需要开太多，默认最大是 20000，但个人用开 50 个就够了
# 因为等会开启 docker 服务之后会监听这里指定的所有端口
# 如果指定了 10000 个，就会开 10000 个监听端口，内存不够可能会撑爆
min-port=10000
max-port=10050

# ssl 证书这里一定要指定为拷贝到 ssl 目录里的对应的证书名
# 包括生成的 dhparams 如果文件名不同也要改
cert=/etc/coturn/ssl/fullchain.pem
pkey=/etc/coturn/ssl/privkey.pem
dh-file=/etc/coturn/ssl/dhparams.pem
```

第八步，拷贝创建 `docker-compose-coturn.yml` 文件，
- 修改 `PUID` 和 `PGID` 
- `WS_FALLBACK` 这个可以不用开，具体原理不清楚，但测试发现在开启了 VPN 的网络里依然是可以加入房间发消息的，只有浏览器插件上明确禁用了 WebRTC 的才会显示无法使用
- 时区也可以修改
- `coturn_server` 那里开启的端口要修改为跟前面配置文件里对应的，也就是 `10000-10050:10000-10050/udp` 

第九步，根据 compose 文件的端口映射，在防火墙开启用到的端口，主要就是 `coturn_server` 用到的那些。

第十步，开启 docker 服务。

此时正常就能通过域名访问 Pairdrop 服务，并且不同网络的设备可以加入公共房间互发消息了。


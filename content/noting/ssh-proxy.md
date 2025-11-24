---
title: SSH 隧道代理
date: 2025-11-24 10:35:00 -0400
tags:
  - ssh
  - proxy
---
## 准备

有一台云服务器，不需要安装 OpenVPN ，其实不需要安装任何东西。SSH 协议天生拥有加密功能。唯一需要注意的是检查一下 SSH 服务端配置文件是否允许转发（默认允许）

```conf title="/etc/ssh/sshd_config"
AllowTcpForwarding yes
```

## 建立隧道

在本地电脑打开终端，使用如下命令

```shell
ssh -D 1080 -N -C username@ip
```

`-D` : 在本地开启端口作为 SOCKS5 代理

`-N` : 不执行远程命令（只转发，不进入 Shell）

`-C` : 开启压缩（可选，网速慢时有用）

输入密码之后终端会卡住不动，这是正常的。但是在之后的使用过程中这个窗口不能关闭。

## 浏览器代理

在浏览器上安装代理插件，选择 SOCKS5 协议，IP 写 `127.0.0.1` ，端口写前面 `-D` 指定的转发端口。这样就可以让浏览器通过 SSH 隧道上网了。

## 终端的一些错误信息

在访问网页时终端可能会出现类似的信息，但是并没有影响网站的正常访问

```txt
channel 6: open failed: connect failed: open failed
```

这通常是因为云服务器先尝试用 IPv6 协议连接目标网站，但是失败了，报错了，之后又尝试用 IPv4 协议连接网站，成功了。

可以尝试下面的命令禁用 IPv6 ，但是也不能完全消除错误信息，总之也不太影响什么

```shell
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
```

## 关于机器人验证

使用代理上网后访问一些网站总弹出来谷歌的或者 Cloudflare 的机器人验证，这是因为开启代理后对于网站来说，它们看到的是来自数据中心的 IP 而不是住宅 IP。正常人肯定都是在家里上网，所以来自数据中心的 IP 会被普遍认为是脚本、爬虫或攻击程序等，信誉度极低。

关于消除这个影响，AI 可能会推荐使用 Cloudflare WARP，但是要注意，**安装之后可能 SSH 就进不去了，慎用**。

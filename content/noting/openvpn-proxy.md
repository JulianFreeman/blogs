---
title: OpenVPN 代理上网
date: 2025-11-23 22:55:00 -0400
tags:
  - vpn
---
## 在云服务器上安装 openvpn3

既然是用 OpenVPN 代理上网了，肯定需要一台不在本地的服务器。本文使用 Ubuntu Server。

手动配置 openvpn 太复杂，搞不懂，所以这里用一个脚本来管理: https://github.com/Nyr/openvpn-install ，按照官方说明下载脚本后，第一次运行会要求创建一个配置文件。之后可以通过再次运行脚本创建新的或者管理已有的配置文件。

> [!info] 说明
> 该脚本也比较简单，不包含用户名密码认证等功能。

> [!remind] 提醒
> 在创建配置文件的时候会要求指定端口，记得设置服务器防火墙放行该端口。

把创建好的配置文件（ .ovpn 文件）下载到家里的本地电脑上备用。

## 全局代理

如果是代理整个电脑的流量，就比较简单了，直接下载 [OpenVPN 客户端](https://openvpn.net/client/) ，然后导入前面下载的配置文件连接就能用了。

但如果想只代理某个特定的浏览器，OpenVPN 官方并没有提供浏览器插件版本，因此无法像 Surfshark VPN 那样通过在某个浏览器上安装官方插件来单独代理这一个浏览器。

所以下面说一个绕弯的办法。

## 在 Ubuntu 上使用 openvpn3

所谓绕弯的办法就是在家里的局域网内找一台 Ubuntu Server，在这个本地服务器上导入 .ovpn 配置文件，然后让本地电脑的浏览器流量先去本地服务器，再由本地服务器通过 OpenVPN 发送给云服务器，然后云服务器连接目的地址。

之所以本地服务器选 Ubuntu（也就是 Linux），一方面方便部署代理服务，另一方面 Linux 上的 openvpn3 支持导入多个配置文件，但是在 Windows 上的 OpenVPN 客户端一次只能连接一个配置文件。

所以在讲方法之前，先说说 openvpn3 在 Ubuntu 上的基本使用。

### 安装

OpenVPN 的 [官方社区](https://community.openvpn.net/Pages/OpenVPN3Linux#stable-repository-debian-ubuntu) 有教程，下面也列举一下：

安装依赖包

```shell
sudo su
apt update
apt install apt-transport-https curl
```

下载签名

```shell
mkdir -p /etc/apt/keyrings
curl -sSfL https://packages.openvpn.net/packages-repo.gpg >/etc/apt/keyrings/openvpn.asc
```

添加 source list ，记得替换 `DISTRIBUTION` ，比如 24.04 是 `noble` 

```shell /DISTRIBUTION/
echo "deb [signed-by=/etc/apt/keyrings/openvpn.asc] https://packages.openvpn.net/openvpn3/debian DISTRIBUTION main" >>/etc/apt/sources.list.d/openvpn3.list
```

安装 openvpn3

```shell
apt update
apt install openvpn3
```

### 基本使用

导入配置文件（开启一个会话）

```shell
openvpn3 session-start --config /path/to/file.ovpn
```

查看所有会话

```shell
openvpn3 sessions-list
```

断开连接

```shell
openvpn3 session-manage --config /path/to/file.ovpn --disconnect
```

把 `disconnect` 改为 `restart` 就是重启会话。

> [!tip] 提示
> 会话是跟用户有关的，比如在 A 用户下创建的会话就只在 A 用户下可见，在 root 用户下创建的会话在 A 用户下就不可见。
> 
> 如果在 A 用户下创建了会话，然后 A 用户注销了，会话也就失效了。因此建议在 root 用户下创建会话。

## 本地服务器安装 dante-server

dante-server 是一个代理服务，我们用它来接受从本地电脑浏览器上发来的流量，然后再转发给对应的 vpn 网卡

```shell
sudo apt update
sudo apt install dante-server
```

但是我们不使用默认服务，所以停掉

```shell
sudo systemctl stop danted.service
sudo systemctl disable danted.service
```

### 编写 dante 配置

创建一个配置目录，名称任意，这里取 `danted.d` ，然后创建第一个配置文件，名称任意，但最后要有端口号

```shell
sudo mkdir /etc/danted.d
cd /etc/danted.d
sudo vim danted-1080.conf
```

```yaml /1080/ /tun0/ /wlo1/ title="/etc/danted.d/danted-1080.conf"
# ================= 日志设置 =================
# 建议指定独立日志文件，方便排查，也可以继续用 syslog
# logoutput: syslog
logoutput: /var/log/danted-1080.log

# ================= 接口设置 =================
internal: wlo1 port = 1080
external: tun0

# ================= 权限与进程管理 =================
# 明确特权用户。
# dante 启动时需要 root 权限来绑定端口和读取系统路由表。
# 启动后，它会自动切换到 user.notprivileged 指定的用户运行子进程。
user.privileged: root
user.notprivileged: nobody

# 进程数量管理
# 默认 dante 可能只开 10 个子进程。如果设备多，并发连接数高，可以适当增加。
# 格式: children: <副本数>
# 10-32 对于家庭使用足够了。
# external.rotation: route # 如果有多条路由策略需要开启这个，单 VPN 不需要。

# ================= 认证方式 =================
# 明确区分 Client 和 Socks 方法
# clientmethod: 谁可以连接到 dante 服务器？(建立 TCP 连接阶段)
# socksmethod: 连接上后，用什么方式协商 SOCKS 协议？(发送请求阶段)
# 在家庭局域网内，都设为 none 是效率最高的。
clientmethod: none
socksmethod: none

# ================= 超时管理 =================
# 防止死连接占用资源
# 协商超时 (默认 30秒，设短点，连不上就赶紧断)
timeout.negotiate: 15
# I/O 超时 (默认 0=无限。设为 0 保持长连接，或者设为一个大数值比如一天)
# 保持 0 适合 SSH 或 挂机下载；设为 3600 适合防止僵尸连接。
timeout.io: 0

# ================= 访问控制规则 (ACL) =================

# 规则 1: 客户端连接控制 (谁能连我)
client pass {
    from: 192.168.0.0/16 to: 0.0.0.0/0
    log: connect error  # 去掉 disconnect，否则日志会非常啰嗦
}

# 规则 2: SOCKS 转发控制 (能去哪里)
socks pass {
    from: 192.168.0.0/16 to: 0.0.0.0/0
    # 协议限制
    # 通常如果你只用浏览器，tcp 就够了。
    # 如果你要玩游戏或者用 HTTP/3 (QUIC)，需要 udp。
    protocol: tcp udp 
    log: connect error
}
```

其中 `wlo1` 是服务器本地上网的网卡，需要根据情况修改； `tun0` 是（通常来说）导入第一个 .ovpn 文件后创建的网卡；绑定的端口是 `1080` 。

因为是从局域网内的本地电脑上接受流量，所以来源那里限定了 `192.168.0.0/16` 。

如果想使用多个配置文件的代理，可以再创建一个 `danted-1081.conf` 文件，基本内容不变，只把网卡改为 `tun1` （通常来说导入第二个配置文件的网卡），然后端口改为 `1081` ，以此类推。

### 编写 dante 服务

创建一个服务文件，名称任意，但一定要有最后的 `@` 符号，内容为

```ini title="/etc/systemd/system/mydanted@.service"
[Unit]
Description=Dante SOCKS proxy instance %i
After=network-online.target
Wants=network-online.target
# 防止 Systemd 认为重启太频繁而彻底放弃该服务
StartLimitIntervalSec=0

[Service]
Type=simple
ExecStart=/usr/sbin/danted -f /etc/danted.d/danted-%i.conf
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

其中 `%i` 会在启动服务时被替换为具体的端口号。

启动服务

```shell
sudo systemctl enable --now mydanted@1080
# 如果有第二个
sudo systemctl enable --now mydanted@1081
```

## 配置路由表和 IP 规则

先说一下手动配置的流程。

### 导入配置文件

> [!warning] 注意
> 在本部分假设使用两个不同的 .ovpn 配置文件，也就是有两个不同的云服务都安装了 OpenVPN，然后通过两个代理配置分别代理到这两个云服务器。

```shell
openvpn3 session-start --config /path/to/file0.ovpn
openvpn3 session-start --config /path/to/file1.ovpn
```

此时对应的网卡为 `tun0` 和 `tun1` ；网卡对应的地址是 `10.8.0.4` 和 `10.8.0.2` 。

### 清除 VPN 的默认路由

导入配置文件后默认路由会被修改为走向这个配置文件创建的网卡。但因为目前本地服务器只是用来代理本地电脑浏览器的流量的，我并不想让整个本地服务器都走 VPN 的流量（否则会导致 `apt update` 也变得很慢），所以这里要清除 VPN 添加的路由，让服务器本身重新通过本地网络联网。

```shell
for dev in tun0 tun1; do
    sudo ip route del 0.0.0.0/1 dev $dev 2>/dev/null
    sudo ip route del 128.0.0.0/1 dev $dev 2>/dev/null
done
```

### 路由和规则配置

添加自定义路由表

```shell
echo "200 vpn0" | sudo tee -a /etc/iproute2/rt_tables
echo "201 vpn1" | sudo tee -a /etc/iproute2/rt_tables
```

为各表添加默认路由

```shell
sudo ip route add default dev tun0 table vpn0
sudo ip route add default dev tun1 table vpn1
```

设置流量来源规则

```shell
sudo ip rule add from 10.8.0.4 table vpn0
sudo ip rule add from 10.8.0.2 table vpn1
```

重启 openvpn 服务

```shell
sudo systemctl restart openvpn.service
```

确保规则和路由都在

```shell
ip rule show
ip route show table vpn0
ip route show table vpn1
```

## 自动化配置路由和规则

上述的路由和规则都是在内存里的，服务器一重启就没了，所以下面提供一个自动化脚本

```shell title="vpn_watchdog.sh"
#!/bin/bash

# ================= 配置区域 =================
# 格式: "配置文件绝对路径  路由表ID  路由表名"
VPN_LIST=(
    "/path/to/file0.ovpn  200  vpn0"
    "/path/to/file1.ovpn  201  vpn1"
)
# ===========================================

if [ "$EUID" -ne 0 ]; then
    echo "请使用 sudo 运行此脚本"
    exit 1
fi

# --- 核心函数：获取 Session 的详细信息 ---
# 返回格式: "SessionPath|StatusString|DeviceName"
get_session_info() {
    local config_path=$1
    
    openvpn3 sessions-list | awk -v cfg="$config_path" '
        # 遇到 Path 行
        /Path:/ {
            if (is_target) { print saved_path "|" saved_status "|" saved_device; exit }
            
            # 重置
            is_target = 0
            saved_status = ""
            saved_device = ""
            
            # 提取 Path (去掉 "Path:" 前后的杂质)
            val = $0; sub(/.*Path:[[:space:]]*/, "", val); sub(/[[:space:]].*/, "", val);
            saved_path = val
        }
        
        # 提取 Status (适配任意缩进)
        /Status:/ { 
            val = $0; sub(/.*Status:[[:space:]]*/, "", val); 
            # Status 后面通常是具体描述，可能包含空格，所以这里只去首尾空格
            sub(/^[[:space:]]*/, "", val); sub(/[[:space:]]*$/, "", val);
            saved_status = val 
        }
        
        # 提取 Device
        /Device:/ { 
            val = $0
            # 1. 无论 Device: 在该行哪里，删掉它以及它之前的所有字符
            sub(/.*Device:[[:space:]]*/, "", val)
            # 2. 删掉设备名(如tun0)后面的空格及其他字段(如果同行还有其他信息)
            sub(/[[:space:]].*/, "", val)
            saved_device = val 
        }
        
        # 检查 Config name
        /Config name:/ { 
            if (index($0, cfg) > 0) { is_target = 1 } 
        }
        
        # 结束块处理
        END {
            if (is_target) { print saved_path "|" saved_status "|" saved_device }
        }
    '
}

# --- 函数：配置路由 ---
configure_routes() {
    local dev=$1
    local table_id=$2
    local table_name=$3

    # 再次清洗 dev 变量，防止任何换行符或残留
    dev=$(echo "$dev" | tr -d '[:space:]')

    if [ -z "$dev" ] || [ "$dev" == "Client" ]; then
        return
    fi

    echo "   -> 配置路由表: $table_name (设备: $dev)"

    if ! grep -q "$table_id $table_name" /etc/iproute2/rt_tables; then
        echo "$table_id $table_name" >> /etc/iproute2/rt_tables
    fi

    ip route del 0.0.0.0/1 dev "$dev" 2>/dev/null || true
    ip route del 128.0.0.0/1 dev "$dev" 2>/dev/null || true

    ip route replace default dev "$dev" table "$table_name"

    local current_ip
    current_ip=$(ip -4 addr show "$dev" | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
    
    if [ -n "$current_ip" ]; then
        while ip rule show | grep -q "lookup $table_name"; do
            ip rule del table "$table_name"
        done
        ip rule add from "$current_ip" table "$table_name"
    fi
}

# --- 主循环 ---
echo "========== [$(date)] VPN 检查启动 =========="

for line in "${VPN_LIST[@]}"; do
    read -r S_CONFIG S_TABLE_ID S_TABLE_NAME <<< "$line"

    echo ">> 检查 Config: $S_CONFIG"

    INFO=$(get_session_info "$S_CONFIG")
    IFS='|' read -r CUR_PATH CUR_STATUS CUR_DEVICE <<< "$INFO"

    NEED_START=0

    # 去除 CUR_DEVICE 可能存在的任何不可见字符
    CUR_DEVICE=$(echo "$CUR_DEVICE" | tr -d '[:space:]')

    if [ -z "$CUR_PATH" ]; then
        echo "   状态: 未运行"
        NEED_START=1
    else
        # 状态检查
        if [[ "$CUR_STATUS" != "Connection, Client connected" ]]; then
            echo "   状态: 异常 [$CUR_STATUS]"
            echo "   操作: 终止僵尸 Session..."
            openvpn3 session-manage --disconnect --path "$CUR_PATH" > /dev/null 2>&1
            sleep 1
            NEED_START=1
        else
            echo "   状态: 健康 (接口: $CUR_DEVICE)"
        fi
    fi

    if [ "$NEED_START" -eq 1 ]; then
        echo "   操作: 启动 Session..."
        openvpn3 session-start --config "$S_CONFIG" > /dev/null 2>&1
        sleep 8
        
        INFO=$(get_session_info "$S_CONFIG")
        IFS='|' read -r CUR_PATH CUR_STATUS CUR_DEVICE <<< "$INFO"
        CUR_DEVICE=$(echo "$CUR_DEVICE" | tr -d '[:space:]')
        
        if [[ "$CUR_STATUS" == "Connection, Client connected" ]]; then
             echo "   结果: 启动成功 ($CUR_DEVICE)"
        else
             echo "   错误: 启动失败 [$CUR_STATUS]"
             continue
        fi
    fi

    configure_routes "$CUR_DEVICE" "$S_TABLE_ID" "$S_TABLE_NAME"

done
echo "=========================================="
```

只需要修改最开始的 `VPN_LIST` ，其他的都自动化处理了。

运行上面的脚本后，再手动重启一下 openvpn 服务

```shell
sudo systemctl restart openvpn.service
```

## 本地电脑浏览器安装代理插件

浏览器需要一个代理插件，来把浏览器的网络流量指向本地服务器上的代理服务（也就是我们自己配置的 dante 服务）。

[Proxy SwitchyOmega](https://chromewebstore.google.com/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif) 在最新的谷歌浏览器上已经用不了了，但在 Brave 浏览器上还能用，当然也可以找其他的替代品。总之只需要在代理插件里选择 SOCKS5 协议，然后指定本地服务器的 IP 地址，以及前面设定的代理服务端口（1080 或者 1081 等），就可以让浏览器走代理了。

访问 https://browserleaks.com/ip 查看 IP 地址以及进行 DNS 泄露测试，只要没有出现本地的公网 IP 就算成功了。

不过还有一点要防的是 WebRTC 泄露 IP 地址，这个可以通过 [WebRTC Control](https://chromewebstore.google.com/detail/webrtc-control/fjkmabmdepjfammlpliljpnbhleegehm) 以及类似插件来控制。

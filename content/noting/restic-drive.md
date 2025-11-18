---
title: 使用 restic 备份到谷歌云盘
date: 2025-11-17 18:22:00 -0400
tags:
  - backup
---
restic 无法直接对接 Google Drive，因此需要借用 rclone。

## 准备工作

1. 一台 Ubuntu Server，这是要备份的文件所在的主机
2. 一个能用 Google Cloud 的谷歌账号（未受限的）
3. 另一台本地电脑，这是登录着谷歌账号的电脑

## 安装软件

在 Ubuntu Server 上安装 restic 和 rclone

```shell
sudo apt update
sudo apt install restic rclone
```

在本地电脑安装 [rclone](https://rclone.org/downloads/) ，同时将执行文件路径添加到环境变量，保证能在终端中直接运行 `rclone` 。这是验证谷歌账号的时候需要的。

## 创建谷歌客户端 ID

这是为了能让 rclone 可以在谷歌云盘里创建和读取文件。可以参考 [官方文档](https://rclone.org/drive/#making-your-own-client-id) ，同时下面也给出具体步骤

> [!info]- 提示
> 客户端 ID 只需创建一次，如果之后有其他主机的 rclone 也想对接到这个谷歌云盘，可以直接复用这个客户端 ID。

1. 访问 https://console.developers.google.com/
2. 左上角新建一个项目，名称任意
3. 切换到新项目，然后点左上角 *启用 API 和服务* 
4. 搜索 drive，选择 Google Drive API，启用
5. 点左侧的 *凭证* ，点顶部右侧的 *配置权限请求页面* 
6. 点 *开始*，应用名称任意，这个是在让谷歌授权时会显示的应用名称
7. 用户支持邮箱选自己的就行，下一步
8. 受众群体选 *外部* ，下一步
9. 联系邮箱填自己的就行，下一步
10. 同意，继续，创建
11. 点左侧 *数据访问*，点 *添加或移除范围*，在下方的 *手动添加范围* 中添加如下三个范围
```text
https://www.googleapis.com/auth/docs,https://www.googleapis.com/auth/drive,https://www.googleapis.com/auth/drive.metadata.readonly
```
12. 点 *添加到表*，看到上面这三条打勾后，点 *更新*，然后点 *Save* 
13. 点左侧 *目标对象*，点下方 *Add users*，把自己的邮箱加进去保存
14. 点左侧 *概览*，点 *创建 OAuth 客户端* 
15. 应用类型选择 *桌面应用*，名称任意，点 *创建* 
16. 保存下来 *客户端 ID* 和 *客户端密钥* 
> [!warning] 客户端密钥只会显示这一次

官方文档最后还发布了应用，但实际上不发布也行，这里就不发布了。

## 配置 rclone 对接 Google Drive

> [!info] 这部分的操作只需要执行一次。

在 Ubuntu Server 上切换为 root 用户，配置文件也会保存在 root 的家目录下，这样 restic 才能备份 root 权限的文件。运行

```shell
sudo su
rclone config
```

接下来的步骤依次是

1. 输入 `n` ，即创建新的远程连接
2. 输入给该远程连接起的名字，比如 `gdrive` 
3. 输入 `drive` ，即在各种支持的远程连接选项中，选择 Google Drive
4. 输入前面创建的客户端 ID
5. 输入前面创建的客户端密钥
6. 输入 `3` ，等同于 `drive.file` ，即权限方面仅允许操作由 rclone 创建的文件
7. 服务账号文件留空，直接回车
8. 输入 `n` ，即不需要高级配置
9. 输入 `n` ，因为我们在远程主机上操作
10. 在本地电脑的终端中运行给出的命令（以 `rclone authorize` 开头的）
11. 确保链接跳转到了前面创建了客户端 ID 的谷歌账号上
12. 进行谷歌验证
13. 验证完成后在终端中会返回一串字符串（通常以 `ey` 开头），复制下来，贴到前面的步骤中
14. 输入 `n` ，即不设置为共享盘
15. 输入 `y` 
16. 输入 `q` ，即退出

## 初始化 restic 仓库

> [!info] 这部分的操作只需要执行一次。

在家目录下创建一个文件，用来保存密码。假设为 `/root/.restic_passwd` 。

运行下面的命令初始化仓库

```shell
restic init -r "rclone:gdrive:test-backup" -p /root/.restic_passwd
```

其中 `-r` 指定仓库路径，这里可以是本地文件系统路径，但因为我们使用的是谷歌云盘，所以这里需要使用 rclone 远程连接的路径，其中 `gdrive` 是在前一部分第二步中设置的连接名称， `test-backup` 是仓库的名字，可以任意。

`-p` 指定密码文件路径

提示创建完成后在谷歌云盘中就能看到这个名为 `test-backup` 的目录了。

因为后续的每次命令操作都需要指定 `-r` 和 `-p` ，为了方便，也可以使用环境变量

```shell
export RESTIC_REPOSITORY="rclone:gdrive:test-backup"
export RESTIC_PASSWORD_FILE="/root/.restic_passwd"
restic init
```

这三行的效果等同于前面的一行，但是在设置好这两个环境变量之后，之后再运行其他 `restic` 命令的时候就不需要附加 `-r` 和 `-p` 参数了。

当然，如果要同时操作多个仓库，那就还是需要手动在每个命令上指定仓库和密码了。

## restic 的常用操作

> [!tip]- 注意
> 运行下面的命令前假设已经配置好了上述两个环境变量。

### 备份目录

```shell
restic backup /path/to/backup
```

### 查看所有快照

```shell
restic snapshots
```

### 恢复快照

```shell
restic restore latest -t /path/to/restore
```

`latest` 表示恢复最新的快照，如果要指定特定的快照，把 `latest` 换成快照 ID。 `-t` 指定要恢复到的路径。

## 创建定时备份任务

### 创建服务文件

```ini title="/etc/systemd/system/restic-backup.service"
[Unit]
Description=Daily Restic Backup

[Service]
Type=oneshot
Environment="RESTIC_REPOSITORY=rclone:gdrive:test-backup"
Environment="RESTIC_PASSWORD_FILE=/root/.restic_passwd"
ExecStart=/usr/bin/restic backup /path/to/backup
ExecStartPost=/usr/bin/restic forget --keep-daily 7 --prune
```

最后一行表示只保留最近 7 天的快照，如果想保留全部，可以去掉最后一行。

### 创建定时器文件

```ini title="/etc/systemd/system/restic-backup.timer"
[Unit]
Description=Run restic backup daily

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

定时器文件会定时触发同名的服务文件。上述时间设置在每天的凌晨 2 点。

### 启动定时器

```shell
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable --now restic-backup.timer
```

## 在另一台电脑上恢复快照

同样在电脑上安装 rclone，然后按照与 [[#配置 rclone 对接 Google Drive]] 相同的步骤设置好远程连接，然后直接用 restic 读取快照就可以了。（仓库是已存在的，所以不需要初始化）

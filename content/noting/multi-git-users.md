---
title: 使用多个 Git 提交身份
date: 2026-04-19 18:36:00 -0400
tags:
  - git
  - ssh
  - gpg
---
## 背景

有时候需要在电脑上使用多个 Git 提交身份提交不同的仓库，比如有的仓库是个人的，需要使用个人的 Github 账号提交，有的仓库是工作上的，需要使用工作上用的 Github 账号提交。

简单的方法就是针对每个仓库设置 `user.email`，但是也许不同的仓库用了不同的 SSH key 或者 GPG key，这个也需要针对仓库设置。 

设置的项目多了，就繁琐，所以下面说个简单的办法。

## 创建 SSH key

使用 Github 的账号邮箱创建一个 SSH key

```sh
ssh-keygen -t ed25519 -C "<YOUR EMAIL>"
```

保存密钥的文件可以改个名，避免用默认的 `id_ed25519` 。

然后把这个 SSH key 添加到 Github 账号上。

## 创建 GPG key

Windows 上安装 Gpg4win 软件，然后命令行可以使用 `gpg` 。

创建一个新的 GPG key

```sh
gpg --full-generate-key
```

算法通常选 RSA and RSA ，密钥长度 4096 ，永不过期。名字和邮箱可以与 Github 账号对应。

创建完成后查看一下 key

```sh
gpg --list-secret-keys --keyid-format=long
```

第一行 `sec  rsa4096/` 之后的部分就是 key 。

导出公钥

```sh
gpg --armor --export <KEY>
```

将公钥添加到 Github 账号。

## 设置不同的 Git 提交者身份

我们可以给不同的仓库分别放在不同的目录下，比如个人 Github 的仓库放在 `D:/Personal/` 目录下，工作 Github 的仓库放在 `D:/Work/` 目录下。

然后修改 `~/.gitconfig` 文件，去掉 `user` 域，添加如下

```ini
[includeIf "gitdir:D:/Personal/"]
    path = ~/.gitconfigd/personal.ini
[includeIf "gitdir:D:/Work/"]
    path = ~/.gitconfigd/work.ini
```

这个意思就是如果仓库在 `D:/Personal/` 目录下，就读取 `~/.gitconfigd/personal.ini` 的配置。另一个同理。

当然这个 `path` 可以随便指定。其文件格式类似

```ini
[core]
	sshCommand = ssh -i ~/.ssh/<YOUR SSH PRIVATE FILE> -o IdentitiesOnly=yes
[user]
	name = <YOUR NAME>
	email = <YOUR EMAIL>
	signingkey = <YOUR GPG KEY>
[commit]
	gpgsign = true
[gpg]
	program = C:\\Program Files\\GnuPG\\bin\\gpg.exe
```

这样不同的仓库就自动使用不同的身份配置了。

不过注意， `gitdir:D:/Personal/` 结尾的 `/` 要有，没有不行。

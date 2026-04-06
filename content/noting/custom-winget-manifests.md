---
title: 制作自定义的 Winget Manifest 文件
date: 2026-04-06 09:20:00 -0400
tags:
  - winget
---

## 背景

Winget 的官方仓库是 https://github.com/microsoft/winget-pkgs ，在 `manifests` 目录下记录着所有能安装的软件，本质其实也是记录了每个软件的安装包下载链接，经过哈希校对之后进行安装。

但问题是，有的软件给的下载链接是不包含版本信息的，比如 ID 是 `Google.Chrome` 的谷歌浏览器，其下载链接是 `https://dl.google.com/dl/chrome/install/googlechromestandaloneenterprise64.msi` ([查看仓库](https://github.com/microsoft/winget-pkgs/blob/master/manifests/g/Google/Chrome/146.0.7680.178/Google.Chrome.installer.yaml)) ，不管软件更新到哪个版本都是这个链接，这就导致一个问题：

某天这个链接指向的执行文件 `googlechromestandaloneenterprise64.msi` 更新到新版了，但是 winget 的仓库的 manifest 文件里对应的哈希没有更新，结果通过 `winget install` 安装的时候出现哈希不匹配安装失败。

## Manifest 文件说明

针对以上问题，我们可以在 `winget install` 的时候指定一个自己写的 manifest 文件，在里面写好下载的安装包的链接以及正确的哈希值，就可以了。

Manifest 文件有几种类型，以 `Google.Chrome` 的 manifest 文件为例，有 `Google.Chrome.yaml` 、 `Google.Chrome.installer.yaml` 、 `Google.Chrome.locale.en-US.yaml` 等。

其中第一个文件是一个类似总述的文件，只包含软件的基本信息，带 `installer` 的是指示如何安装的文件，带 `locale` 的是记录不同语言版本的信息。这里面真正起到安装作用的就是带 `installer` 文件的信息。

但是我们也不能直接把 `installer` 文件拿过来换个链接和哈希就完事了，因为一般 `installer` 文件的信息是不完整的，我们自己自定义的 manifest 文件可以不用整那么多分文件，官方也支持一种 `singleton` 类型的文件，即在一个文件中包含所有必要的信息。

## 制作 Manifest 文件

先把要自定义 manifest 文件的软件的原版 installer 文件拿过来，把标头的 `yaml-language-server` 改为：

```yaml
# yaml-language-server: $schema=https://aka.ms/winget-manifest.singleton.1.12.0.schema.json
```

这个必须要有，表示当前文件使用 singleton 的 schema 。

然后我们打开 https://aka.ms/winget-manifest.singleton.1.12.0.schema.json 这个链接，找到最后可以看到这个 schema 中哪些信息是必须要包含的：

```json
{
    ...
    "required": [
        "PackageIdentifier",
        "PackageVersion",
        "PackageLocale",
        "Publisher",
        "PackageName",
        "License",
        "ShortDescription",
        "Installers",
        "ManifestType",
        "ManifestVersion"
    ]
}
```

然后把这些信息从软件原版的几个 manifest 文件中照抄到我们自定义的 manifest 文件中即可。因为我自定义的文件是以 installer 文件为基础的，所以一般已经包含 `PackageIdentifier` 、 `PackageVersion` 、 `Installers` 等信息了。

**但是 `ManifestType` 这个要改成 `singleton` ，因为我们用的就是单一文件的类型** 。 `ManifestVersion` 的版本只要跟 schema 的版本一致即可。

还有这个 `Installers` 中可能包含不同架构的安装包，可以根据自己的需要删减。最后把安装包的链接和哈希(SHA256)换掉，这个 manifest 文件就制作完成了。

## 使用 Manifest 文件

要想在 `winget install` 的时候使用自定义的 manifest 文件，我们需要先在 **管理员终端** 运行这个命令：

```cmd
winget settings --enable LocalManifestFiles
```

然后使用如下命令安装：

```cmd
winget install --manifest /path/to/manifest.yaml
```

不过注意，这个 manifest 文件只能是本地文件，不能是网络文件。

如果因为 manifest 内容错误无法安装，可以使用如下命令检查哪里有问题：

```cmd
winget validate --manifest /path/to/manifest.yaml
```

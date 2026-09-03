---
title: 'Mac Antigravity IDE 无法连接 OrbStack x86_64 Ubuntu 26.04 LTS'
date: 2026-09-03T10:42:29+08:00
draft: false
description: 解决 Apple Silicon Mac 上 Antigravity 通过 SSH 连接 OrbStack x86_64 Linux 时，GNU tar 报 Function not implemented 导致远程服务器安装失败的问题。
categories: ["mac"]
tags: ["antigravity", "orbstack", "linux"]
slug: "antigravity-orbstack"
cover:
---

## 问题

Mac 上使用 Antigravity IDE 通过 SSH 远程连接到 OrbStack 中启动的 x86_64 Ubuntu 26.04 LTS 虚拟机时，卡在 `Setting up SSH Host: Launching SSH server...`

部分日志输出如下：
``` text
tar: ./out/server-cli.js: Cannot open: Function not implemented 
tar: ./out/bootstrap-fork.js: Cannot open: Function not implemented 
tar: ./out/server-main.js: Cannot open: Function not implemented
```

Apple Silicon Mac 上运行 amd64/x86_64 OrbStack VM，并使用 Rosetta 执行 x86_64 Linux 程序时，较新版本的 GNU tar（如 1.35）在解压包含多级目录的压缩包时，可能出现 `Function not implemented` 错误[^1]。Antigravity 在远程安装 Server 时需要使用 `tar` 解压，因此安装过程失败并一直停留在 `Launching SSH server...`。

[^1]: https://github.com/orbstack/orbstack/issues/2588

## 解决方案

安装 `bsdtar`，并在 `/usr/local/bin` 中创建名为 `tar` 的软链接，使其通过 `PATH` 优先于系统自带的 `/usr/bin/tar`。

{{< tabs >}}
{{< tab label="安装" >}}
```bash
sudo apt update
sudo apt install libarchive-tools

sudo ln -s /usr/bin/bsdtar /usr/local/bin/tar
```
{{< /tab >}}
{{< tab label="验证" >}}
``` bash
which tar
tar --version
```
{{< /tab >}}
{{< tab label="期望结果" >}}
``` text
/usr/local/bin/tar
bsdtar x.x.x
```
{{< /tab >}}
{{< /tabs >}}

重新通过 Antigravity 连接虚拟机即可。

后续如果需要使用原本的 `tar`，删除软连接就可以恢复。
``` bash
sudo rm /usr/local/bin/tar
```
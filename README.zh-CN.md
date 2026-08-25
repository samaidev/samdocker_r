# SamDocker — 把废旧安卓手机变成可远程控制的 Linux 服务器

一个 Android APK，装到旧手机上之后，手机插上充电器丢抽屉里 = 一台 7×24 在线的、可远程访问的 Linux 服务器。

[English version](./README.md)

---

## 这是什么

安装 SamDocker APK 到旧手机后：

1. **开机自启**一个前台 Service
2. 在手机里跑一个 **proot Linux 环境**（Alpine，约 5MB）—— 无需 root
3. 自动启动 **基于浏览器的终端**（xterm.js）
4. 通过 **aitun.cc Quick Tunnel** 暴露成一个随机子域名
5. **每次启动重新生成 token**，URL 形如：
   ```
   https://random-words-xyz.t.aitun.cc/?token=8f3a2b9c1e...
   ```
6. 用电脑浏览器打开这个地址，输入 token，就能得到一个完整的 Linux shell（`ls`、`cd`、`vim`、`apt`、`git`、`python3`、`gcc` 都能用）

## 功能特性

- **无需 root** — 通过 `proot` 在用户空间运行
- **完整 Alpine Linux** —— 预装 `bash`、`git`、`vim`、`curl`、`python3`、`gcc`、`openssh`、`tmux`
- **原生 git** —— clone、pull、fetch、commit、push 全部支持（私有仓库嵌入 token 即可）
- **web terminal 支持元字符** —— `&&`、`;`、`|`、`>`、`<` 都能正常工作
- **文件管理器** —— 上传、下载、浏览工作目录文件
- **状态持久化** —— `apk add` 装的包重启后仍在
- **OpenRC + `systemctl` 兼容层** —— 习惯 systemd 的用户也能直接用
- **`apt`/`apt-get` 兼容层** —— 自动映射到 `apk`
- **自动重连看门狗** —— 隧道断开会自动重连
- **电池优化绕过** —— 屏幕关闭也能继续跑

## 安装方法

1. 从 [最新 release](https://github.com/samaidev/samdocker_r/releases/latest) 下载 `samdocker.apk`
2. 把 APK 传到安卓手机（USB、蓝牙、浏览器都可以）
3. 在手机文件管理器里点击 APK 安装
   - 如果系统提示，先在 Android 设置里开启「允许从未知来源安装应用」
4. 打开 **SamDocker** 应用
5. 首次启动等待 30–90 秒（一次性初始化：解压 rootfs、装包）
6. 屏幕会显示一个 URL，形如：
   ```
   https://random-words-xyz.t.aitun.cc/?token=...
   ```
7. 在任意浏览器打开这个 URL，输入 token，即可获得 Linux shell

**最低要求：** Android 6.0（API 23）或更高，ARM 32 位或 64 位。

## 终端使用示例

连上后就是一个真实的 Alpine Linux shell。常用命令：

```bash
# 安装包（Alpine 原生）
apk add nginx python3 py3-pip

# 或用 apt 兼容层
apt install htop

# git clone（私有仓库需在 URL 里嵌入 token）
git clone https://github.com/octocat/Hello-World.git
git clone https://USER:TOKEN@github.com/yourname/yourrepo.git

# 命令链也支持
cd /tmp && git clone https://github.com/octocat/Hello-World.git && cd Hello-World && git log --oneline -3

# 服务管理（通过 systemctl 兼容层 → openrc）
systemctl start sshd
systemctl enable sshd
```

### 长任务（训练、服务、监控）

任何超过 2 分钟的任务都应该跑在**后台**，否则 tunnel 超时会被杀。在 web terminal 里用 `bg:` 前缀：

```
bg: make train
bg: python3 long_train.py
bg: ./serve.sh
```

会启动一个 samcommand 后台任务 — 你会看到 job ID，并提示点击终端右上角的 **⚡ Jobs** 按钮查看实时输出、终止任务、查看退出码。任务会在 tunnel 断连后继续跑，没有时间限制。

### 编译 / 构建原生代码

`gcc`、`g++`、`make`、`libgomp` (OpenMP)、`openblas-dev` 都已预装。`/etc/profile.d/samdocker.sh` 默认设了 `CFLAGS=-O2 -fno-strict-aliasing -fopenmp -funroll-loops` —— 避开 `-march=native` 和 `-ffast-math` 在 proot 下可能触发的 SIGSEGV。

```bash
git clone https://github.com/your/repo.git && cd repo
make            # 直接能跑
./binary        # 跑产物
```

如果你 `apk add` 装了带 hardlink 的包（gcc、binutils 等），apk wrapper 会自动修复失效的 hardlink —— 不需要手动 `ln -sf`。

## 源码

- **App + 构建脚本**：[samaidev/samdocker](https://github.com/samaidev/samdocker)（源码、构建文档、架构说明）
- **Release**：本仓库（samdocker_r）—— 预编译 APK 直接下载
- **内嵌的终端服务器**：[samaidev/samcommand](https://github.com/samaidev/samcommand)（Go 二进制，提供 web terminal）

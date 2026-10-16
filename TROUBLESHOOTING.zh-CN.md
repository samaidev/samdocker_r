# 故障排查

## `bash: applet not found` —— 智能体 / 工具报"bash 用不了"

**影响版本：** SamDocker v2.1.18（versionCode 20118），rootfs 构建日期 2025-08-26。更新的 rootfs 构建可能不受影响。

### 现象

在 SamDocker 的 Linux 环境里，任何显式调用 `bash` 的操作都会失败：

```
$ bash --version
bash: applet not found
$ bash -c "echo hello"
bash: applet not found        （退出码 127）
```

带 bash 工具的 AI 智能体（one-api / OneAPI Dashboard 的智能体抽屉、各种 agent harness、`#!/bin/bash` 开头的脚本）会报告"bash 用不了"，但 `which bash` 又确实能找到 `/bin/bash`。

交互式 `sh` 使用不受影响——busybox 的 `sh` 能处理普通 POSIX 命令，所以这个问题平时很难察觉，直到智能体真的去调 `bash` 才爆发。

### 根因

APK 自带的 rootfs 镜像里塞了一个**假的 `/bin/bash`**：其实是 busybox 二进制改名的拷贝（硬链接），而包数据库里真 `bash` 包仍显示"已安装"。以下是在一台 v2.1.18 真机上采集的证据：

| 检查项 | 结果 |
|---|---|
| `md5sum /bin/bash /bin/busybox` | **完全相同** —— 均为 `3b168fdd100cfa204df26c4ad642d507` |
| `ls -li /bin/bash /bin/busybox` | 同一 inode `4454494`（硬链接对） |
| `apk info -e bash` / `apk info -L bash` | 已安装 `bash-5.2.26-r0`，文件清单声称 `bin/bash` |
| `bash --version` | `bash: applet not found`（busybox 报错：没有 `bash` 这个 applet） |
| `/bin/sh` | 符号链接 → `/bin/bash`（即同样指向那份 busybox 拷贝） |

也就是说：`bash` 经 PATH 查找到的是一个 busybox 拷贝，busybox 没有 `bash` applet，所有 `bash` 调用均以 127 退出。`apk` 认为已安装的真 GNU bash 5.2.26 在 rootfs 镜像构建时就被覆盖掉了。

**这个文件在环境内部无法被替换。** 实测以下操作全部失败，而同目录其他文件操作正常：

- `apk fix bash` → `failed to rename bin/.apk.XXX to bin/bash`（rename 被拒）
- `cat 真bash > /bin/bash` → `can't create /bin/bash: Permission denied`（写打开被拒）
- `rm /bin/bash` / `mv 覆盖` → `Permission denied`（unlink 被拒）
- `chattr -i /bin/bash` → `Permission denied`

对照组：在 `/bin` 里新建文件正常；删除**其他**预置文件（如 `/etc/motd`）也正常。被拒的只有 `/bin/bash`–`/bin/busybox` 这对 inode，非 root 身份无法原地修复（proot 假 root 没有 `CAP_LINUX_IMMUTABLE`；底层存储是应用私有数据目录）。

同一区域还有两个次要问题，排查时值得知道：

1. REST / web 终端的 exec 会话会把 APK 自带的 busybox 绑定挂到 `/bin/sh`、`/bin/ash`、`/bin/busybox` 上（proot 命令行可见），并用 `env -i` 启动命令——因此 `SHELL` 变量为空，`/bin/sh` 恒为 busybox，与 rootfs 里写什么无关。
2. exec 的硬编码 `PATH` 是 `/bin:/usr/bin:/sbin:/usr/sbin:/usr/local/bin`，`/bin` 排最前——所以装到 `/usr/bin/bash` 或 `/usr/local/bin/bash` 的真 bash **不会**在普通 `bash` 查找中遮住坏的那个。

### 诊断一行命令

```sh
md5sum /bin/bash /bin/busybox   # 哈希相同 ⇒ 中招
bash --version                  # "applet not found" ⇒ 中招
```

### 绕过方案（按进程生效，无需 root）

通过 apk 的备用 root 装出真 bash，放进私有目录，再为需要 bash 的进程把该目录前置到 `PATH`：

```sh
# 1. 在备用 root 里物化真 bash（不碰被锁死的文件）
apk add --initdb -p /tmp/broot \
    --keys-dir /etc/apk/keys \
    -X https://mirrors.aliyun.com/alpine/v3.20/main bash

# 2. 放到持久位置
mkdir -p ~/bashbin
cp /tmp/broot/bin/bash ~/bashbin/bash
chmod 755 ~/bashbin/bash

# 3. 用前置 PATH 启动受影响的进程
PATH="$HOME/bashbin:$PATH" <你的智能体或守护进程>
```

用 `bash -c 'echo $BASH_VERSION'` 验证——应输出 `5.2.26(1)-release...`。

注意：PATH 前置只存在于用它启动的那个进程的环境里。应用完全重启 / 服务自动恢复后，进程可能不带前置 PATH 回来——需要重新应用，或改用始终 `export PATH` 的启动脚本。

### 建议的源码级修复（rootfs 构建）

- rootfs 镜像必须在 `/bin/bash` 放真 GNU bash 二进制；不要用 busybox 拷贝/硬链接占位。
- 镜像组装后在构建机上断言：`md5sum rootfs/bin/bash` 与 `rootfs/bin/busybox` 不同，且 `rootfs/bin/bash` 不是指向 busybox 的硬链接。
- 仅当真 bash 存在时才保留 `/bin/sh → /bin/bash`；否则把 `/bin/sh` 直接指向 busybox，别走假链条。
- 建议在 exec 环境里导出 `SHELL=/bin/bash`（当前因 `env -i` 为空）。

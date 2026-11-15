# 故障排查

## `bash: applet not found` — AI 代理/工具报告 "bash 不可用"

**受影响版本：** SamDocker v2.1.18（versionCode 20118），rootfs 构建于 2025-08-26。较新的 rootfs 可能不受影响。

### 症状

在 SamDocker Linux 环境中，任何显式调用 `bash` 的操作都会失败：

```
$ bash --version
bash: applet not found
$ bash -c "echo hello"
bash: applet not found        (退出码 127)
```

带有 `bash` 工具的 AI 代理（one-api / OneAPI Dashboard 代理、带有 `#!/bin/bash` shebang 的脚本）报告 bash "不可用"，尽管 `which bash` 能找到 `/bin/bash`。

交互式 `sh` 使用不受影响，因为 busybox `sh` 能处理普通 POSIX 命令——这使得问题在代理实际调用 `bash` 之前很容易被忽略。

### 根本原因

APK 附带的 rootfs 镜像包含一个**假的 `/bin/bash`**：busybox 二进制文件以 bash 名称复制（硬链接），而包数据库仍然将真正的 `bash` 包列为已安装。

| 检查 | 结果 |
|---|---|
| `md5sum /bin/bash /bin/busybox` | **相同** — `3b168fdd100cfa204df26c4ad642d507` |
| `ls -li /bin/bash /bin/busybox` | 相同 inode `4454494`（硬链接对） |
| `apk info -e bash` | 已安装，`bash-5.2.26-r0` |
| `bash --version` | `bash: applet not found`（busybox 没有 bash applet） |

### 诊断一行命令

```sh
md5sum /bin/bash /bin/busybox   # 哈希相同 ⇒ 受影响
bash --version                  # "applet not found" ⇒ 受影响
```

### 解决方案（无需 root）

将真正的 bash 安装到私有目录，然后将其加入 PATH：

```sh
mkdir -p ~/bashbin
apk add --root ~/bashbin --initdb --no-cache bash
export PATH=~/bashbin/usr/bin:$PATH
bash --version   # GNU bash, version 5.2.26(1)-release
```

永久修复：将 `export PATH` 行添加到 `~/.profile` 或 `~/.bashrc`。

---

## 后台进程被杀/冻死——息屏后进程不响应（Android 12+）

**受影响版本：** 所有 SamDocker 版本，Android 12（API 31）及以上。Android 13–16 尤为严重。

### 症状

在 SamDocker Linux 环境中启动的长时间运行进程（Web 服务器、守护进程、训练任务）在手机息屏一段时间后静默死亡或冻僵。进程可能仍出现在 `ps` 中（状态 `S`，wchan `futex_wait_queue_me`），但不再响应网络请求。

### 根本原因

Android 12+ 引入了**幻影进程杀手（Phantom Process Killer）**，会积极杀掉消耗过多 CPU/内存的 App 子进程。在 proot 中启动的进程被视为 SamDocker App 的子进程——成为首要目标。

此外，**Doze 模式**限制未在电池优化白名单中的 App 的网络访问，导致隧道和 HTTP 服务器即使进程还活着也停止响应。

### 修复方法（需要 ADB，一次性设置）

将手机连接到有 ADB 的电脑，执行以下三条命令：

```bash
# 1. 禁用幻影进程杀手（最关键！）
adb shell settings put global settings_enable_monitor_phantom_procs false

# 2. 将 SamDocker 加入 Doze 白名单（防止网络冻结）
adb shell dumpsys deviceidle whitelist +com.samaidev.samdocker

# 3. 禁用 SamDocker 的电池优化
adb shell dumpsys deviceidle whitelist +com.samaidev.samdocker
```

**第 1 步是关键修复。** 它禁用杀 App 子进程的系统服务。此设置重启后仍然有效，但 Android 大版本更新后可能被重置——如果更新后进程又开始被杀，重新执行即可。

### 验证

应用修复后，启动一个长时间运行的进程，息屏 30 分钟以上再检查。进程应该仍然存活并响应：

```sh
# 在 SamDocker 终端中
curl -s http://127.0.0.1:3000/   # 应该仍然响应
```

### 没有 ADB 的替代方案

如果没有带 ADB 的电脑，可以通过手机设置部分缓解：

1. **设置 → 应用 → SamDocker → 电池 → 不受限制**（防止 Doze 网络限制）
2. **设置 → 开发者选项 → 后台进程限制 → 标准**（或"不超过 1 个进程"改为"标准"）
3. 保持手机插着充电器（充电状态会降低 Doze 的攻击性）

注意：幻影进程杀手无法在没有 ADB 的情况下禁用——它没有 UI 开关。如果进程持续被杀且没有 ADB，唯一的解决方案是设置看门狗自动重启关键服务（SamDocker 内置的 JobMonitor 会为 `~/.config/samdocker/jobs/` 中带 `# Binary:` 指令的作业执行此操作）。

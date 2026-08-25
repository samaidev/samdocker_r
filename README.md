# SamDocker — Turn an Old Android Phone into a Remote Linux Server

An Android APK that turns a discarded phone into a 24/7 cloud-accessible Linux server. Install the APK, plug the phone into a charger, drop it in a drawer — you now have a remote Linux shell reachable from any browser.

[中文版 (Chinese version)](./README.zh-CN.md)

---

## What It Does

After installing SamDocker on an Android phone:

1. A **foreground Service** auto-starts on boot
2. A **proot Linux environment** (Alpine, ~5 MB) runs inside Android — no root required
3. A **browser-based terminal** (xterm.js) is served over HTTPS
4. An **aitun.cc Quick Tunnel** exposes it on a random subdomain
5. A **fresh token is generated on every launch** — the URL looks like:
   ```
   https://random-words-xyz.t.aitun.cc/?token=8f3a2b9c1e...
   ```
6. Open the URL on any device, enter the token, and you get a full Linux shell (`ls`, `cd`, `vim`, `apt`, `git`, `python3`, `gcc` …)

Effect: a junk-drawer phone + a USB cable = a 24/7 remote Linux server.

## Features

- **No root required** — runs entirely in user space via `proot`
- **Full Alpine Linux** with `bash`, `git`, `vim`, `curl`, `python3`, `gcc`, `openssh`, `tmux`
- **Native `git`** — clone, pull, fetch, commit, push all work (with private-repo token support)
- **Shell metacharacters** in the web terminal — `&&`, `;`, `|`, `>`, `<` all supported
- **File manager** — upload, download, browse files in the working directory
- **Persistent state** — packages installed via `apk add` survive across reboots
- **OpenRC + `systemctl` wrapper** — service management feels familiar to systemd users
- **`apt`/`apt-get` compatibility layer** — maps to `apk` for Debian muscle memory
- **Auto-reconnect watchdog** — if the tunnel drops, it reconnects automatically
- **Battery optimization bypass** — keeps running when the screen is off

## Install

1. Download `samdocker.apk` from the [latest release](https://github.com/samaidev/samdocker_r/releases/latest)
2. Transfer it to your Android phone (USB, Bluetooth, browser — whatever works)
3. Tap the APK in your file manager to install
   - You may need to enable "Install from unknown sources" in Android Settings first
4. Open the **SamDocker** app
5. Wait 30–90 seconds on first launch (one-time setup: extracting rootfs, installing packages)
6. The screen will display a URL like:
   ```
   https://random-words-xyz.t.aitun.cc/?token=...
   ```
7. Open that URL in any browser, enter the token, and you have a Linux shell

**Minimum requirements:** Android 6.0 (API 23) or newer, ARM 32-bit or 64-bit.

## Upgrading

New releases of SamDocker can be installed on top of old ones — your installed packages, files, and tokens persist. Just download the new `samdocker.apk` and tap install; Android will replace the old version.

## Reporting Issues

Please open an issue at https://github.com/samaidev/samdocker_r/issues with:
- Phone model and Android version
- SamDocker version (shown on the app's main screen)
- Steps to reproduce
- Logcat snippet if possible

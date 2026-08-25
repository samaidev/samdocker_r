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

## Using the Terminal

Once connected, you have a real Alpine Linux shell. Common commands:

```bash
# Install packages (Alpine native)
apk add nginx python3 py3-pip

# Or use the apt compatibility wrapper
apt install htop

# Git clone (private repos need token in URL)
git clone https://github.com/octocat/Hello-World.git
git clone https://USER:TOKEN@github.com/yourname/yourrepo.git

# Shell chains work too
cd /tmp && git clone https://github.com/octocat/Hello-World.git && cd Hello-World && git log --oneline -3

# Service management (via systemctl wrapper → openrc)
systemctl start sshd
systemctl enable sshd
```

### Long-running tasks (training, servers, watchers)

Anything that takes longer than ~2 minutes should run in the **background**, otherwise the tunnel times out and the task gets killed. Use the `bg:` prefix in the web terminal:

```
bg: make train
bg: python3 long_train.py
bg: ./serve.sh
```

This starts a samcommand background job — you'll see the job ID and a tip to click the **⚡ Jobs** button (top-right of the terminal) to view live output, kill the job, or check its exit code. Jobs survive tunnel disconnects and run indefinitely.

### Compile / build native code

`gcc`, `g++`, `make`, `libgomp` (OpenMP), and `openblas-dev` are preinstalled. Default `CFLAGS` are set in `/etc/profile.d/samdocker.sh` to `-O2 -fno-strict-aliasing -fopenmp -funroll-loops` — this avoids proot SIGSEGV that `-march=native` and `-ffast-math` can trigger.

```bash
git clone https://github.com/your/repo.git && cd repo
make            # works out of the box
./binary        # run the result
```

If you `apk add` any package with hardlink-based binaries (gcc, binutils, etc.), the apk wrapper auto-repairs the broken hardlinks — no manual `ln -sf` needed.

## Source Code

- **App + build scripts**: [samaidev/samdocker](https://github.com/samaidev/samdocker) (source code, build docs, architecture)
- **Releases**: this repo (samdocker_r) — prebuilt APKs for direct download
- **Embedded terminal server**: [samaidev/samcommand](https://github.com/samaidev/samcommand) (Go binary that serves the web terminal)

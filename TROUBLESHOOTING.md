# Troubleshooting

## `bash: applet not found` — agents / tools report "bash is unavailable"

**Affected:** SamDocker v2.1.18 (versionCode 20118), rootfs built 2025-08-26. Newer rootfs builds may not be affected.

### Symptom

Anything that explicitly spawns `bash` fails inside the SamDocker Linux environment:

```
$ bash --version
bash: applet not found
$ bash -c "echo hello"
bash: applet not found        (exit code 127)
```

AI agents that ship a `bash` tool (one-api / OneAPI Dashboard agent drawer, agent harnesses, scripts with `#!/bin/bash` shebangs) report that bash "cannot be used", even though `which bash` finds `/bin/bash`.

Interactive `sh` usage is unaffected because busybox `sh` handles plain POSIX commands — which makes the breakage easy to miss until an agent actually invokes `bash`.

### Root cause

The rootfs image shipped with the APK contains a **fake `/bin/bash`**: the busybox binary copied (hardlinked) under the bash name, while the package database still lists the real `bash` package as installed. Evidence collected from a live v2.1.18 device:

| Check | Result |
|---|---|
| `md5sum /bin/bash /bin/busybox` | **identical** — `3b168fdd100cfa204df26c4ad642d507` both |
| `ls -li /bin/bash /bin/busybox` | same inode `4454494` (hardlink pair) |
| `apk info -e bash` / `apk info -L bash` | installed, `bash-5.2.26-r0`, file list says `bin/bash` |
| `bash --version` | `bash: applet not found` (busybox error: no `bash` applet) |
| `/bin/sh` | symlink → `/bin/bash` (i.e. → the same busybox copy) |

So: `bash` resolves through PATH to a busybox copy, busybox has no `bash` applet, every `bash` invocation exits 127. The real GNU bash 5.2.26 that `apk` believes is installed was overwritten in the rootfs image at build time.

**The file cannot be replaced from inside the environment.** Verified attempts all fail while sibling files behave normally:

- `apk fix bash` → `failed to rename bin/.apk.XXX to bin/bash` (rename over the file is denied)
- `cat real-bash > /bin/bash` → `can't create /bin/bash: Permission denied` (open-for-write denied)
- `rm /bin/bash` / `mv over it` → `Permission denied` (unlink denied)
- `chattr -i /bin/bash` → `Permission denied`

Controls that pass: creating new files in `/bin` works, and deleting *other* pre-existing files (e.g. `/etc/motd`) works. The denial is specific to the `/bin/bash`–`/bin/busybox` inode pair, so in-place repair is blocked for non-root users (proot fake-root has no `CAP_LINUX_IMMUTABLE`; the backing store is the app's private data dir).

Two secondary quirks in the same area, worth knowing when debugging:

1. REST/web-terminal exec sessions bind the APK's own busybox over `/bin/sh`, `/bin/ash` and `/bin/busybox` (visible in the proot command line), and launch commands with `env -i` — so `SHELL` is empty and `/bin/sh` is busybox regardless of what the rootfs says.
2. Because `/bin` precedes every other dir in the hardcoded exec `PATH` (`/bin:/usr/bin:/sbin:/usr/sbin:/usr/local/bin`), a real bash installed to `/usr/bin/bash` or `/usr/local/bin/bash` does **not** shadow the broken one for plain `bash` lookups.

### Diagnosis one-liner

```sh
md5sum /bin/bash /bin/busybox   # identical hashes ⇒ affected
bash --version                  # "applet not found" ⇒ affected
```

### Workaround (per process, no root needed)

Install real bash into a private dir via an apk side-root, then prepend that dir to `PATH` for the process that needs bash:

```sh
# Install real bash into ~/bashbin
mkdir -p ~/bashbin
apk add --root ~/bashbin --initdb --no-cache bash
# Now use it:
export PATH=~/bashbin/usr/bin:$PATH
bash --version   # GNU bash, version 5.2.26(1)-release
```

For a permanent fix, add the `export PATH` line to `~/.profile` or `~/.bashrc`.

---

## Background processes killed / frozen when screen is off (Android 12+)

**Affected:** All SamDocker versions on Android 12 (API 31) and newer. Especially severe on Android 13–16.

### Symptom

Long-running processes started inside the SamDocker Linux environment (web servers, daemons, training jobs) silently die or freeze when the phone screen is off for a while. The process may still appear in `ps` (state `S`, wchan `futex_wait_queue_me`) but stops responding to network requests.

### Root cause

Android 12+ introduced the **Phantom Process Killer** (phantom process monitoring), which aggressively kills app child processes that consume too much CPU/memory over time. Processes started inside proot are treated as child processes of the SamDocker app — making them prime targets.

Additionally, **Doze mode** restricts network access for apps not on the battery optimization whitelist, causing tunnels and HTTP servers to stop responding even if the process is still alive.

### Fix (requires ADB, one-time setup)

Connect the phone to a computer with ADB and run these three commands:

```bash
# 1. Disable Phantom Process Killer (most important!)
adb shell settings put global settings_enable_monitor_phantom_procs false

# 2. Add SamDocker to Doze whitelist (prevents network freeze)
adb shell dumpsys deviceidle whitelist +com.samaidev.samdocker

# 3. Disable battery optimization for SamDocker
adb shell dumpsys deviceidle whitelist +com.samaidev.samdocker
```

**Step 1 is the critical fix.** It disables the system service that kills app child processes. This setting persists across reboots but may be reset after a major Android system update — re-run it if processes start dying again after an OS update.

### Verification

After applying the fixes, start a long-running process, turn off the screen, and check back after 30+ minutes. The process should still be alive and responding:

```sh
# Inside SamDocker terminal
curl -s http://127.0.0.1:3000/   # should still respond
```

### Without ADB

If you don't have a computer with ADB, you can partially mitigate via phone settings:

1. **Settings → Apps → SamDocker → Battery → Unrestricted** (prevents Doze network restrictions)
2. **Settings → Developer Options → Background process limit → Standard** (or "No limit")
3. Keep the phone plugged into a charger (charging state reduces Doze aggression)

Note: The Phantom Process Killer cannot be disabled without ADB — it has no UI toggle. If processes keep dying and you don't have ADB, the only workaround is to set up a watchdog that auto-restarts critical services (SamDocker's built-in JobMonitor does this for jobs with `# Binary:` directives in `~/.config/samdocker/jobs/`).

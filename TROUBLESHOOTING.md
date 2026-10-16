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
# 1. materialize real bash in an alternative root (no touching the locked file)
apk add --initdb -p /tmp/broot \
    --keys-dir /etc/apk/keys \
    -X https://mirrors.aliyun.com/alpine/v3.20/main bash

# 2. keep it somewhere persistent
mkdir -p ~/bashbin
cp /tmp/broot/bin/bash ~/bashbin/bash
chmod 755 ~/bashbin/bash

# 3. start the affected process with bashbin first in PATH
PATH="$HOME/bashbin:$PATH" <your-agent-or-daemon>
```

Verify with `bash -c 'echo $BASH_VERSION'` — it should print `5.2.26(1)-release...`.

Caveat: the PATH prepend only lives in the environment of the process you start it with. After a full app restart / service auto-recovery, the process may come back without the prepend — re-apply it, or use a launcher script that always exports `PATH`.

### Recommended source-level fix (rootfs build)

- The rootfs image must ship the real GNU `bash` binary at `/bin/bash`; never copy/hardlink busybox over it, even as a placeholder.
- After image assembly, assert on the build host: `md5sum rootfs/bin/bash` differs from `rootfs/bin/busybox`, and `rootfs/bin/bash` is not a hardlink to busybox.
- Keep `/bin/sh → /bin/bash` only if the real bash is present; otherwise point `/bin/sh` at busybox directly instead of the fake chain.
- Consider exporting `SHELL=/bin/bash` in the exec environment (currently empty due to `env -i`).

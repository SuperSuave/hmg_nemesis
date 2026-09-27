# NEMESIS packages

NEMESIS is the head unit’s **Debugging Execution Interface**. A **package** is a zip you install through that interface so tools, daemons, and optional APKs live under `/data/nemesis/apps/<id>/` and can be started, launched, or removed without flashing the system image again.

This document assumes NEMESIS is already on the unit. USB install of a zip uses `NEMESIS.data` with `type=install` (see [05](05-Run-Scripts.md) for execute vs install). Opening Manager is `type=open` ([02](02-Debugging-Execution-Interface.md)).

---

## 1. What a package is

A zip whose **root** contains **`NEMESIS.data`** (INI). The builder copies the whole directory tree into the zip, paths relative to the package folder.

Installed layout:

```text
/data/nemesis/apps/<id>/          unpacked tree
/data/nemesis/packages.db         id|name|locked|types
/data/nemesis/services.db         registered services
```

The locked core package **`usb-handler`** is seeded by the service manager. It cannot be updated or uninstalled except by wiping NEMESIS.

Same `id` already installed → **update** (Yes/No in Manager when Android is up; automatic when Android is down). There is never a second copy of the same id.

---

## 2. `NEMESIS.data` — `[nemesis]`

| Key | Required | Meaning |
|-----|----------|---------|
| `id` | yes | Install id. Max 64 characters: letters, digits, `.`, `_`, `-`. No leading `.`, no `..`. |
| `name` | recommended | Display name (defaults to `id`) |
| `version` | recommended | Free-form version string |
| `types` | recommended | Comma list, e.g. `application,service` (shown in `pkg-list`) |
| `preinstall` | no | Script run **before** the tree is copied, cwd = staging dir |
| `postinstall` | no | Script run **after** copy, APK install, and service register; cwd = `/data/nemesis/apps/<id>` |
| `preuninstall` | no | Script run after services are stopped, before APKs/`rm` |
| `postuninstall` | no | Copied to tmp then run so it still exists after the tree is deleted |

All listed scripts run as **root** (or `system` if you set `uid=system` on a service; hooks themselves use root).

---

## 3. Applications — `[app.N]`

Zero or more sections `[app.0]`, `[app.1]`, …

| Key | Meaning |
|-----|---------|
| `kind` | `sh` — launchable script. `apk` — file to `pm install -r` at install time. |
| `path` | Relative to the package root (`bin/tool.sh`) or, for uninstall of APKs, the Android package name you passed to `pm` |
| `uid` | `root` (default) or `system` |

`nemesis pkg-launch <id>` runs the first `kind=sh` app script.

`kind=apk`: the file at `path` is `pm install -r` during install. Uninstall calls `pm uninstall` with that `path` string — put the **Android package name** in `path` if you rely on uninstall, or accept that `pm` may be missing if zygote is down.

---

## 4. Services — `[service.N]`

Sections `[service.0]`, `[service.1]`, … Registry name is **`<package-id>/<name>`** so two packages can both use `name=daemon`.

| Key | Meaning |
|-----|---------|
| `name` | Local name (required) |
| `kind` | `sh` (default) — `/system/bin/sh` on `path`. `instance` — `am start` / `am startservice` on `path` (needs Android). |
| `path` | Script relative to the package, or activity/service component for `instance` |
| `uid` | `root` (default) or `system` (uid/gid 1000) |
| `mode` | `restart` — svcmgr restarts when the pid dies. Ordinary packages stop after 5 rapid deaths; the locked `usb-handler` is always restarted. `background` — start once, no restart. `once` — wait for exit. |

`kind=instance` is deferred while Android is down. `kind=sh` does not need zygote.

Stdout/stderr of `sh` services go to `/data/nemesis/logs/<id>/<name>.log` (survives reboot).

CLI:

```sh
nemesis service list
nemesis service start hello.tool/hello-daemon
nemesis service stop hello.tool/hello-daemon
nemesis service status hello.tool/hello-daemon
nemesis log service hello.tool/hello-daemon
```

---

## 5. How to create a package

### 5.1 Directory

```text
my-tool/
  NEMESIS.data
  bin/tool.sh          # optional app
  svc/daemon.sh        # optional service
  scripts/pre.sh
  scripts/post.sh
  scripts/preun.sh
  scripts/postun.sh
  app/Something.apk    # optional
```

Minimal `NEMESIS.data`:

```ini
[nemesis]
version=1.0
id=my.tool
name=My Tool
types=application,service
preinstall=scripts/pre.sh
postinstall=scripts/post.sh
preuninstall=scripts/preun.sh
postuninstall=scripts/postun.sh

[app.0]
kind=sh
path=bin/tool.sh
uid=root

[service.0]
name=my-daemon
kind=sh
path=svc/daemon.sh
uid=root
mode=restart
```

Scripts must be `#!/system/bin/sh` and toolbox-only. Prefer `nemesis remount rw` / `nemesis remount ro` to change `/system`. Toolbox `mount -o remount,rw /system` returns `Operation not permitted` on this unit even as uid 0 (the kernel refuses a remount that would drop `nosuid`). Hooks and `nemesis root` put `/data/nemesis/bin` first on `PATH`, so a bare `mount -o remount,rw /system` in a hook is wrapped. You can still `export PATH=/sbin:/vendor/bin:/system/sbin:/system/bin:/system/xbin` for other tools.

Worked examples in the source tree: `examples/package-hello/`, `examples/package-hello-v2/` (`mode=background` on v2), `packages/usb-handler/`.

### 5.2 Zip

```bash
python3 tools/make_nemesis_package.py /path/to/my-tool -o my.tool.zip
```

That zips every file under the directory. Inspect with `nemesis pkg-info my.tool.zip` on the unit.

### 5.3 Install on the unit

**USB (usual):**

```ini
[nemesis]
version=1.0
type=install
path=packages/my.tool.zip
```

USB root:

```text
NEMESIS.data
packages/my.tool.zip
```

`path` must be relative (no `/`, no `..`, no shell metacharacters). Insert the stick. If Android is up, confirm install. If the id exists, you get an update prompt.

**CLI:**

```sh
nemesis install /storage/usb0/packages/my.tool.zip
nemesis install --update /storage/usb0/packages/my.tool.zip
```

**Manager:** Packages list after USB or CLI install; Launch / Uninstall there.

### 5.4 Uninstall

```sh
nemesis uninstall my.tool
```

Locked packages print `package is locked` and stay. Wipe NEMESIS removes everything, including locked.

---

## 6. Install order (what the CLI does)

1. Read `NEMESIS.data` from the zip; reject bad `id`.
2. Refuse if locked and already installed.
3. Require `--update` (or USB update confirm) if the id exists.
4. Extract to `/data/local/tmp/nemesis/stage/<id>` (zip-slip `..` members are not applied).
5. `preinstall` → copy tree to `/data/nemesis/apps/<id>/` → `pm install` APKs → register services → write `packages.db` → `postinstall` → start services.

Logs: `nemesis log dump` and `/data/nemesis/nemesis.log`.

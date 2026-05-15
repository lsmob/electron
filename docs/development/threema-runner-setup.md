# Threema CI — Self-Hosted Runner Setup

This document explains how to set up self-hosted GitHub Actions runners for the
[`threema-build.yml`](../../.github/workflows/threema-build.yml) workflow, which
builds a custom Electron with Threema's WebRTC patches.

The workflow is triggered manually via **Actions → Run workflow**. You select which
platforms to build each time; nothing runs automatically on push.

---

## Hardware requirements

| Platform | Runner labels | Disk | RAM | Notes |
|---|---|---|---|---|
| Linux x64 | `self-hosted, linux, x64` | 200 GB+ | 16 GB+ | Runs inside Docker container |
| macOS arm64 | `self-hosted, macOS, X64` | 200 GB+ | 16 GB+ | Cross-compiled on Intel host |
| macOS x64 | `self-hosted, macOS, X64` | 200 GB+ | 16 GB+ | Native on Intel host |
| Windows x64 | `self-hosted, Windows, x64` | 200 GB+ | 16 GB+ | Native Windows |

> macOS arm64 and macOS x64 share the same Intel runner. Run them as separate
> workflow dispatches, not simultaneously.

---

## GitHub runner registration (all platforms)

Before setting up any runner, generate a registration token:

1. Go to `https://github.com/lsmob/electron/settings/actions/runners`
2. Click **New self-hosted runner**
3. Select the OS and architecture
4. Copy the token — it expires in **1 hour**

At the end of setup each runner should appear as **Idle** in the Runners list with
the correct labels.

---

## Linux x64

The build runs inside `ghcr.io/electron/build` (Electron's official Docker image),
so the host only needs Docker and the runner agent.

### 1 — Install Docker

```bash
curl -fsSL https://get.docker.com | sh
```

### 2 — Create a dedicated user

```bash
sudo useradd -m -s /bin/bash github-runner
sudo usermod -aG docker github-runner
```

### 3 — Download and configure the runner

```bash
sudo -u github-runner -i

mkdir -p /data/actions-runner   # choose a path with 200 GB+ free
cd /data/actions-runner

# Download — use the exact URL shown in the GitHub UI for the current runner version
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v<VERSION>/actions-runner-linux-x64-<VERSION>.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

./config.sh \
  --url https://github.com/lsmob/electron \
  --token <TOKEN> \
  --name linux-builder \
  --labels self-hosted,linux,x64 \
  --work /data/actions-runner/_work \
  --unattended
```

### 4 — Install as a systemd service

```bash
exit   # back to admin user
cd /data/actions-runner
sudo ./svc.sh install github-runner
sudo ./svc.sh start
sudo ./svc.sh status
```

Check logs:
```bash
journalctl -u actions.runner.lsmob-electron.linux-builder -f
```

### 5 — Pre-pull the build image

Avoids a slow cold pull on the first workflow run:

```bash
sudo -u github-runner docker pull \
  ghcr.io/electron/build:daad061f4b99a0ae1c841be4aa09188280a9c8a4
```

---

## macOS (Intel — arm64 cross-compile and x64 native)

macOS builds run natively without Docker. The `fix-sync` action installs
platform-specific toolchain binaries (clang, gn, ninja, siso) after `gclient sync`.

### 1 — Install Xcode

Install Xcode from the App Store (full Xcode, not just Command Line Tools).
After installing, accept the license:

```bash
sudo xcodebuild -license accept
```

### 2 — Install Homebrew and dependencies

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git node python3
```

### 3 — Download and configure the runner

```bash
mkdir -p ~/actions-runner
cd ~/actions-runner

# Download — use the exact URL shown in the GitHub UI
curl -o actions-runner-osx-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v<VERSION>/actions-runner-osx-x64-<VERSION>.tar.gz
tar xzf actions-runner-osx-x64.tar.gz

./config.sh \
  --url https://github.com/lsmob/electron \
  --token <TOKEN> \
  --name macos-intel-builder \
  --labels self-hosted,macOS,X64 \
  --work ~/actions-runner/_work \
  --unattended
```

### 4 — Install as a launchd service

```bash
./svc.sh install
```

```bash
# Allow the agent to run without a GUI session
/usr/libexec/PlistBuddy \
  -c "Add :LimitLoadToSessionType string Background" \
  ~/Library/LaunchAgents/actions.runner.lsmob-electron.macos-intel-builder.plist

# Start — use user/ domain, not gui/ (gui/ requires an active GUI session)
launchctl bootstrap user/$(id -u) \
  ~/Library/LaunchAgents/actions.runner.lsmob-electron.macos-intel-builder.plist

# Stop
launchctl bootout user/$(id -u) \
  ~/Library/LaunchAgents/actions.runner.lsmob-electron.macos-intel-builder.plist

# Status
launchctl print user/$(id -u)/actions.runner.lsmob-electron.macos-intel-builder
```

> **Note:** `svc.sh install` regenerates the plist each time, so re-apply the
> `PlistBuddy` command after any reinstall.

Check logs:
```bash
tail -f ~/Library/Logs/actions.runner.lsmob-electron.macos-intel-builder/Runner_*.log
```

### Notes

- **arm64 build**: the workflow cross-compiles from this Intel host using
  `target_cpu="arm64"`. No additional setup required.
- **x64 build**: native compilation, also on this Intel host. It is disabled by
  default in the workflow since arm64 covers the primary target; enable it
  explicitly when needed.
- The first sync downloads ~30 GB of Chromium source. Subsequent runs reuse
  whatever Chromium source is already on disk from prior workflow runs.

---

## Windows x64

The build runs natively on Windows. `depot_tools` downloads and manages the MSVC
toolchain automatically — no manual Visual Studio installation needed.

### 1 — Enable long paths

Open PowerShell as Administrator:

```powershell
Set-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
  -Name LongPathsEnabled -Value 1
```

### 2 — Install Git for Windows

Download from https://git-scm.com/download/win. During setup, enable:
- **Git Credential Manager**
- **Enable long paths** (if not already set via registry)

### 3 — Install Node.js

Download the LTS installer from https://nodejs.org.

### 4 — Download and configure the runner

Open PowerShell as the user that will run builds (not Administrator):

```powershell
New-Item -ItemType Directory -Path D:\actions-runner   # choose a drive with 200 GB+
Set-Location D:\actions-runner

# Download — use the exact URL shown in the GitHub UI
Invoke-WebRequest `
  -Uri https://github.com/actions/runner/releases/download/v<VERSION>/actions-runner-win-x64-<VERSION>.zip `
  -OutFile actions-runner-win-x64.zip
Expand-Archive actions-runner-win-x64.zip -DestinationPath .

.\config.cmd `
  --url https://github.com/lsmob/electron `
  --token <TOKEN> `
  --name windows-builder `
  --labels self-hosted,Windows,x64 `
  --work D:\actions-runner\_work `
  --unattended
```

### 5 — Install as a Windows service

```powershell
.\svc.cmd install
.\svc.cmd start
.\svc.cmd status
```

Check logs in `D:\actions-runner\_diag\`.

### Notes

- `depot_tools` downloads the pinned MSVC toolchain during `fix-sync` — this is a
  multi-GB download on the first run.
- Antivirus scanning of the build directory can cause significant slowdowns. Add
  `D:\actions-runner\_work` to your antivirus exclusion list.

---

## Secrets

Set these in `https://github.com/lsmob/electron/settings/secrets/actions`:

| Secret | Required | Purpose |
|---|---|---|
| `ELECTRON_RBE_JWT` | Optional | Enables Electron's siso remote build cache. When absent, all compilation happens locally on the runner. |
| `CHROMIUM_GIT_COOKIE` | Optional | Raises Chromium git server rate limits. Builds work without it but may be throttled during `gclient sync`. |
| `CHROMIUM_GIT_COOKIE_WINDOWS_STRING` | Optional | Same as above, Windows format. |

---

## Consuming the builds in Threema Desktop

Successful builds are published as a GitHub Release tagged `v<version>-threema`.
Configure `@electron/get` in Threema Desktop to download from there:

```bash
ELECTRON_MIRROR=https://github.com/lsmob/electron/releases/download/
ELECTRON_CUSTOM_DIR=v{{ version }}-threema
```

Or in `package.json` under `build.electronDownload` (electron-builder):

```json
"electronDownload": {
  "mirror": "https://github.com/lsmob/electron/releases/download/",
  "customDir": "v{{ version }}-threema"
}
```

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
After installing, accept the license and download the Metal shader toolchain
(split out from Xcode since Xcode 14):

```bash
sudo xcodebuild -license accept
sudo xcodebuild -downloadComponent MetalToolchain
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

On macOS Ventura and later, `svc.sh start` is broken (`launchctl load` is
deprecated). The generated plist also lacks a `LimitLoadToSessionType`, which
causes bootstrap to fail without a GUI session. Fix both issues and raise the
file descriptor limits so the build doesn't need `sudo` at runtime:

```bash
PLIST=~/Library/LaunchAgents/actions.runner.lsmob-electron.macos-intel-builder.plist

# Allow the agent to run without a GUI session
/usr/libexec/PlistBuddy -c "Add :LimitLoadToSessionType string Background" "$PLIST"

# Raise file descriptor limits — avoids sudo during the build
/usr/libexec/PlistBuddy -c "Add :SoftResourceLimits dict" "$PLIST"
/usr/libexec/PlistBuddy -c "Add :SoftResourceLimits:NumberOfFiles integer 10000" "$PLIST"
/usr/libexec/PlistBuddy -c "Add :HardResourceLimits dict" "$PLIST"
/usr/libexec/PlistBuddy -c "Add :HardResourceLimits:NumberOfFiles integer 65536" "$PLIST"
```

```bash
# Start — use user/ domain, not gui/ (gui/ requires an active GUI session)
launchctl bootstrap user/$(id -u) "$PLIST"

# Stop
launchctl bootout user/$(id -u) "$PLIST"

# Status
launchctl print user/$(id -u)/actions.runner.lsmob-electron.macos-intel-builder
```

> **Note:** `svc.sh install` regenerates the plist each time, so re-apply the
> `PlistBuddy` commands after any reinstall.

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

The build runs natively on Windows. `depot_tools` downloads the MSVC toolchain
for the Chromium/Electron build, but Visual Studio Build Tools must also be
installed separately so that node-gyp can compile Electron's native test
fixtures during `yarn install`.

### 1 — Enable long paths and script execution

Open PowerShell as Administrator:

```powershell
# Allow long file paths (required for Chromium source tree)
Set-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
  -Name LongPathsEnabled -Value 1

# Allow PowerShell scripts to run (required by the runner and depot_tools)
Set-ExecutionPolicy Bypass -Scope LocalMachine -Force
```

### 2 — Install Git for Windows

Download from https://git-scm.com/download/win. During setup, enable:
- **Git Credential Manager**
- **Enable long paths** (if not already set via registry)

After installation, add Git's `bin` directory to the **system** PATH so that
`bash.exe` is available to the runner service (the installer only adds `cmd\`
by default). Run this in an elevated PowerShell, then restart the service so
it picks up the new PATH:

```powershell
$gitBin = "C:\Program Files\Git\bin"
$currentPath = [System.Environment]::GetEnvironmentVariable("PATH", "Machine")
[System.Environment]::SetEnvironmentVariable("PATH", "$currentPath;$gitBin", "Machine")
Restart-Service "actions.runner.*"
```

### 3 — Install Node.js

Download the LTS installer from https://nodejs.org.

The installer adds Node.js to the **user** PATH only. The runner service runs
as `NT AUTHORITY\NETWORK SERVICE` and uses the **system** PATH. Add both the
Node.js directory and the `NETWORK SERVICE` npm prefix to system PATH in an
elevated PowerShell:

```powershell
# C:\Program Files\nodejs       — node.exe and npm.cmd
# NetworkService AppData\npm    — globally installed CLI tools installed by the
#                                 runner service (e.g. `e` from @electron/build-tools)
$nodePath    = "C:\Program Files\nodejs"
$npmPrefix   = "C:\Windows\ServiceProfiles\NetworkService\AppData\Roaming\npm"
# depot_tools is installed here by install-build-tools on the first run.
# Adding it upfront makes python3.bat visible to PowerShell steps in fix-sync.
$depotTools  = "C:\Windows\ServiceProfiles\NetworkService\.electron_build_tools\third_party\depot_tools"
$currentPath = [System.Environment]::GetEnvironmentVariable("PATH", "Machine")
[System.Environment]::SetEnvironmentVariable("PATH", "$currentPath;$nodePath;$npmPrefix;$depotTools", "Machine")
```

> The `npm` prefix is `%APPDATA%\npm` evaluated as the service account, which
> resolves to the `NetworkService` profile — not to any user's home directory.

### 4 — Install Visual Studio Build Tools

Required by node-gyp to compile Electron's native test fixtures during
`yarn install`. Download and run the bootstrapper (~3 GB, 10–15 min):

```powershell
Invoke-WebRequest -Uri "https://aka.ms/vs/17/release/vs_buildtools.exe" `
  -OutFile "$env:TEMP\vs_buildtools.exe"

& "$env:TEMP\vs_buildtools.exe" --quiet --wait --norestart `
  --add Microsoft.VisualStudio.Workload.VCTools `
  --includeRecommended
```

If the install completes but node-gyp still reports **"missing any VC++ toolset"**,
the workload was registered without the compiler. Fix it using the VS Installer
that was already placed on disk:

```powershell
Start-Process -Wait `
  -FilePath "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\setup.exe" `
  -ArgumentList 'modify --installPath "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools" --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 --includeRecommended --quiet --norestart'
```

Alternatively open the Visual Studio Installer GUI (`setup.exe` above without
arguments), click **Modify** on the BuildTools entry and tick
**Desktop development with C++**.

node-gyp finds MSVC automatically via the registry — no PATH changes or
runner restart needed.

### 5 — Download and configure the runner

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
  --unattended `
  --runasservice
```

### 6 — Start the Windows service

The runner is registered as a Windows service automatically by `config.cmd` — there is
no separate install step. Manage it with PowerShell (run as Administrator):

```powershell
# Start
Start-Service "actions.runner.*"

# Check status
Get-Service "actions.runner.*" | Select-Object Name, Status, StartType

# Stop
Stop-Service "actions.runner.*"
```

The service is configured to start automatically on boot. You can also manage it via
the Windows **Services** application (`services.msc`).

Check logs in `D:\actions-runner\_diag\`.

### Notes

- `depot_tools` downloads the pinned MSVC toolchain during `fix-sync` — this is a
  multi-GB download on the first run.
- Antivirus scanning of the build directory can cause significant slowdowns. Add
  `D:\actions-runner\_work` to your antivirus exclusion list.
- The runner service must be started **after** all system PATH changes are made
  (Git `bin\`, Node.js, and npm prefix). The service captures PATH at startup
  and does not pick up changes until it is restarted.

---

## Secrets

Set these in `https://github.com/lsmob/electron/settings/secrets/actions`:

| Secret | Required | Purpose |
|---|---|---|
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

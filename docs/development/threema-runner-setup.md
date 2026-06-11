# Threema CI — Self-Hosted Runner Setup

This document explains how to set up self-hosted GitHub Actions runners for the
[`threema-build.yml`](../../.github/workflows/threema-build.yml) workflow, which
builds a custom Electron with Threema's WebRTC patches.

The workflow is triggered manually via **Actions → Run workflow**. You select which
platforms to build each time; nothing runs automatically on push.

---

## Hardware requirements

| Runner | Labels | CPU | RAM (testing) | RAM (release) | Disk |
|---|---|---|---|---|---|
| Linux x64 | `self-hosted, linux, x64` | 16+ cores | 32 GB | 64 GB | 400 GB SSD |
| Linux arm64 | `self-hosted, linux, x64` | 16+ cores | 32 GB | 64 GB | 400 GB SSD |
| macOS arm64 | `self-hosted, macOS, X64` | 8+ cores (Intel) | 32 GB | 64 GB | 400 GB SSD |
| macOS x64 | `self-hosted, macOS, X64` | 8+ cores (Intel) | 32 GB | 64 GB | 400 GB SSD |
| Windows x64 | `self-hosted, Windows, x64` | 32 cores | 64 GB | 128 GB | 200 GB boot + 600 GB data SSD |

---

## GitHub runner registration (all platforms)

Before setting up any runner, generate a registration token:

1. Go to `https://github.com/threema-ch/electron/settings/actions/runners`
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

mkdir -p /home/github-runner/actions-runner   # choose a path with 200 GB+ free
cd /home/github-runner/actions-runner

# Download — use the exact URL shown in the GitHub UI for the current runner version
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v<VERSION>/actions-runner-linux-x64-<VERSION>.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

./config.sh \
  --url https://github.com/threema-ch/electron \
  --token <TOKEN> \
  --name linux-builder \
  --labels self-hosted,linux,x64 \
  --work /home/github-runner/actions-runner \
  --unattended
```

### 4 — Install as a systemd service

```bash
exit   # back to admin user
cd /home/github-runner/actions-runner
sudo ./svc.sh install github-runner
sudo ./svc.sh start
sudo ./svc.sh status
```

Check logs:
```bash
journalctl -u actions.runner.threema-ch-electron.linux-builder -f
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

**Requirements:** macOS 12 or later, Xcode (latest), Node.js 22.12.0+, Python 3.9+.

### 1 — Install Xcode

Install Xcode from the App Store (full Xcode, not just Command Line Tools).
After installing, accept the license and download the Metal shader toolchain
(split out from Xcode since Xcode 14):

```bash
sudo xcodebuild -license accept
sudo xcodebuild -downloadComponent MetalToolchain
```

`xcodebuild -downloadComponent` only downloads the toolchain DMG — it must be
installed manually. Mount the downloaded DMG and copy the toolchain into Xcode:

```bash
DMG=$(find /System/Library/AssetsV2/com_apple_MobileAsset_MetalToolchain \
  -name "*.dmg" | head -1)
hdiutil attach "$DMG" -mountpoint /tmp/metal-toolchain
sudo cp -a /tmp/metal-toolchain/Metal.xctoolchain \
  /Applications/Xcode.app/Contents/Developer/Toolchains/
hdiutil detach /tmp/metal-toolchain
```

Verify:
```bash
xcrun --toolchain Metal metal --version
```

The workflow sets `TOOLCHAINS=Metal` so the build picks it up automatically.

### 2 — Install Homebrew and dependencies

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git node python3
```

Verify minimum versions after install:

```bash
node --version   # must be >= 22.12.0
python3 --version  # must be >= 3.9
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
  --url https://github.com/threema-ch/electron \
  --token <TOKEN> \
  --name macos-intel-builder \
  --labels self-hosted,macOS,X64 \
  --work ~/actions-runner/_work \
  --unattended
```

### 4 — Install as a launchd service

Run from an active GUI session (logged-in desktop, not SSH):

```bash
./svc.sh install
./svc.sh start
./svc.sh status
```

```bash
# Stop
./svc.sh stop

# Uninstall (before re-registering the runner)
./svc.sh uninstall
```

> **Surviving reboots:** Enable Automatic Login (**System Settings → Users & Groups →
> Automatic Login → select the runner user**). Without it the launchd agent won't load
> until someone logs in via GUI, leaving the runner offline after every reboot.
> Automatic Login is unavailable when FileVault is enabled or when the account password
> is managed via iCloud — you may need to disable FileVault on a dedicated CI machine.

Check logs:
```bash
tail -f ~/Library/Logs/actions.runner.threema-ch-electron.macos-intel-builder/Runner_*.log
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

### 1 — System configuration

Open PowerShell as Administrator:

```powershell
# Long file paths — required for Chromium source tree
Set-ItemProperty `
  -Path "HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem" `
  -Name LongPathsEnabled -Value 1

# Script execution — required by the runner and depot_tools
Set-ExecutionPolicy Bypass -Scope LocalMachine -Force

# Exclude the build work directory from Windows Defender
Add-MpPreference -ExclusionPath "D:\actions-runner\_work"
```

### 2 — Install Git and Node.js

Download and install both:
- **Git for Windows** from https://git-scm.com/download/win — during setup enable **Git Credential Manager** and **Enable long paths**
- **Node.js** LTS (v22.12.0 or later) from https://nodejs.org

Both installers add their binaries to the **user** PATH only. Add them to the
**system** PATH so the runner service can find them:

```powershell
$gitBin   = "C:\Program Files\Git\bin"   # bash.exe for runner scripts
$nodePath = "C:\Program Files\nodejs"
$cur = [System.Environment]::GetEnvironmentVariable("PATH", "Machine")
[System.Environment]::SetEnvironmentVariable("PATH", "$cur;$gitBin;$nodePath", "Machine")
```

### 3 — Create the runner user

The runner service runs as a dedicated local account. `config.cmd` grants it
`SeServiceLogonRight` automatically during registration.

```powershell
net user github-runner <PASSWORD> /add
```

Create the runner directory and grant the account full access:

```powershell
New-Item -ItemType Directory -Path D:\actions-runner
icacls "D:\actions-runner" /grant "github-runner:(OI)(CI)F"
```

Write the Git config into the account's profile. Windows only creates the profile
directory on first login, so create it first. `install-build-tools` only sets
these for MSYS2 bash; Git for Windows is silently skipped otherwise:

```powershell
New-Item -ItemType Directory -Force -Path "C:\Users\github-runner"
$cfg = "C:\Users\github-runner\.gitconfig"
git config --file $cfg core.filemode          false
git config --file $cfg core.autocrlf          false
git config --file $cfg core.fscache           true
git config --file $cfg core.longpaths         true
git config --file $cfg core.preloadindex      true
git config --file $cfg branch.autosetuprebase always
```

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
that was placed on disk during the previous step:

```powershell
Start-Process -Wait `
  -FilePath "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\setup.exe" `
  -ArgumentList 'modify --installPath "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools" --add Microsoft.VisualStudio.Component.VC.Tools.x86.x64 --includeRecommended --quiet --norestart'
```

node-gyp finds MSVC automatically via the registry — no PATH changes or
runner restart needed.

### 5 — Add Debugging Tools for Windows

Required for release builds to generate PDB files for crash reporting:

```powershell
Invoke-WebRequest -Uri "https://go.microsoft.com/fwlink/?linkid=2164145" `
  -OutFile "$env:TEMP\winsdksetup.exe"
& "$env:TEMP\winsdksetup.exe" /features OptionId.WindowsDesktopDebuggers /quiet /norestart
```

### 6 — Download and configure the runner

```powershell
Set-Location D:\actions-runner

# Download — use the exact URL shown in the GitHub UI
Invoke-WebRequest `
  -Uri https://github.com/actions/runner/releases/download/v<VERSION>/actions-runner-win-x64-<VERSION>.zip `
  -OutFile actions-runner-win-x64.zip
Expand-Archive actions-runner-win-x64.zip -DestinationPath .

.\config.cmd `
  --url https://github.com/threema-ch/electron `
  --token <TOKEN> `
  --name windows-builder `
  --labels self-hosted,Windows,x64 `
  --work D:\actions-runner\_work `
  --unattended `
  --runasservice `
  --windowslogonaccount ".\github-runner" `
  --windowslogonpassword "<PASSWORD>"
```

### 7 — Start the Windows service

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

The service starts automatically on boot. Check logs in `D:\actions-runner\_diag\`.

### Notes

- `depot_tools` downloads the pinned MSVC toolchain during `fix-sync` — this is a
  multi-GB download on the first run. It installs into the `github-runner` user
  profile and is added to PATH by `fix-sync` within the build environment.
- The runner service must be started **after** all system PATH changes are made.
  The service captures PATH at startup and does not pick up changes until restarted.

---

## Secrets

Set these in `https://github.com/threema-ch/electron/settings/secrets/actions`:

| Secret | Required | Purpose |
|---|---|---|
| `CHROMIUM_GIT_COOKIE` | Optional | Raises Chromium git server rate limits. Builds work without it but may be throttled during `gclient sync`. |
| `CHROMIUM_GIT_COOKIE_WINDOWS_STRING` | Optional | Same as above, Windows format. |
```

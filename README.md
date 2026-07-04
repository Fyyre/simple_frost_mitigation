# Chrome OPFS RAM Mitigation (Poor Man's FROST Defense)

**Goal**: Protect against the **[FROST](https://hannesweissteiner.com/pdfs/frost.pdf)** side-channel attack by keeping the Origin Private File System (OPFS) in RAM while leaving the rest of the Chrome profile on disk.

## Mitigation Strategies Overview

### Mitigation Strat 1: Move Profile to Mechanical Hard Drive + Symlink (Simplest) 😼
- Move the entire Chrome `User Data` folder (or just the profile) to your mechanical HDD.
- Create a symbolic link back to the original location.
- **Pros**: Zero RAM usage, very simple, excellent FROST protection (no SSD contention).
- **Cons**: Slower browser performance (HDD I/O).
- **How-to**:
  1. Copy `C:\Users\YourName\AppData\Local\Google\Chrome\User Data` to HDD (e.g. `D:\ChromeProfile`).
  2. Delete the original folder.
  3. Run as Admin:
     ```cmd
     mklink /J "C:\Users\YourName\AppData\Local\Google\Chrome\User Data" "D:\ChromeProfile"
     ```
  4. Launch Chrome normally.

This is often the easiest "set and forget" option if you have a mechanical drive.

---

### Mitigation Strat 2: Targeted OPFS in RAM (Recommended Hybrid)

**Features**
- Minimal RAM usage (~1 GB)
- Full protection against FROST
- Works with your workstation lock / PIN login workflow
- Automatic setup on login + save on logoff

## Prerequisites & ImDisk Installation

1. Download **ImDisk Toolkit** → [https://sourceforge.net/projects/imdisk-toolkit/](https://sourceforge.net/projects/imdisk-toolkit/)
2. Install it (Run as Administrator).

1. Download **ImDisk Toolkit** from the official source:  
   [https://sourceforge.net/projects/imdisk-toolkit/](https://sourceforge.net/projects/imdisk-toolkit/)

2. Install it (recommended: **Run as Administrator** during install).

3. After installation, verify `imdisk.exe` is available in your PATH or in `C:\Program Files\ImDisk\`.

- Run all batches **as Administrator**

## Setup

### 1. Create Backup Folder
```cmd
mkdir C:\RAMBackups\ChromeOPFS
```

### 2. Setup Batch (`Setup_Chrome_OPFS_RAM.bat`)

```batch
@echo off
:: Setup OPFS symlink to RAM
:: Run as Administrator

set RAMDRIVE=R:
set RAMSIZE=1024M
set PROFILE_PATH=%LOCALAPPDATA%\Google\Chrome\User Data\Default
set OPFS_TARGET=%RAMDRIVE%\OPFS
set BACKING=C:\RAMBackups\ChromeOPFS

echo === Chrome OPFS RAM Setup ===

:: Create RAM drive if needed
if not exist %RAMDRIVE%\ (
    echo Creating RAM drive...
    imdisk -a -s %RAMSIZE% -m %RAMDRIVE% -p "/fs:ntfs /q /y"
)

:: Ensure target folders
if not exist "%BACKING%" mkdir "%BACKING%"
if not exist "%OPFS_TARGET%" mkdir "%OPFS_TARGET%"

:: Check for existing symlink first
if exist "%PROFILE_PATH%\File System" (
    echo Symlink or folder already exists.
    if not exist "%PROFILE_PATH%\File System\*" (
        echo Existing symlink detected.
    ) else (
        echo Real folder detected - moving to RAM...
        xcopy "%BACKING%\" "%OPFS_TARGET%\" /E /H /C /I /Y
        rd /s /q "%PROFILE_PATH%\File System"
        mklink /J "%PROFILE_PATH%\File System" "%OPFS_TARGET%"
        echo Symlink created.
    )
) else (
    echo Creating new symlink...
    mklink /J "%PROFILE_PATH%\File System" "%OPFS_TARGET%"
    echo Symlink created.
)

echo Setup complete.
pause
```

### 3. Save Batch (`Save_Chrome_OPFS.bat`) — Optional but recommended before shutdown

```batch
@echo off
:: Save Chrome RAM profile back to disk

set RAMDRIVE=R:
set BACKING=C:\RAMBackups\ChromeProfile

echo Saving Chrome profile from RAM to %BACKING%...

if exist %RAMDRIVE%\ChromeProfile\ (
    xcopy "%RAMDRIVE%\ChromeProfile\*" "%BACKING%\" /E /H /C /I /Y /D
    echo Sync completed successfully.
) else (
    echo RAM profile not found!
)

pause
```

## Automation

### Auto-Run Setup on Login (Task Scheduler)
1. Open **Task Scheduler** → Create Task.
2. **General** tab: Name it "Setup Chrome OPFS RAM", check **Run with highest privileges**.
3. **Triggers** tab:
   - New → **At log on**
   - Specific user: Select your username
   - Enabled: Checked
4. **Actions** tab:
   - New → **Start a program**
   - Program/script: `C:\Windows\System32\cmd.exe`
   - Add arguments: `/c start "" "C:\Path\To\Your\Setup_Chrome_OPFS_RAM.bat"`
5. Save the task.

### Auto-Save on Logoff
1. Create another task named "Save Chrome OPFS".
2. **General** tab: **Run with highest privileges**.
3. **Triggers** tab:
   - New → **On an event**
   - Basic
   - Log: **Security**
   - Source: **Microsoft-Windows-Eventlog**
   - Event ID: **4647**
4. **Actions** tab:
   - Program/script: `C:\Windows\System32\cmd.exe`
   - Add arguments: `/c start "" "C:\Path\To\Your\Save_Chrome_OPFS.bat"`

## Usage
- The login task will auto-mount the RAM disk + symlink.
- The logoff task will save OPFS data.
- Use Chrome normally.

## Usage
- Run `Setup_Chrome_OPFS_RAM.bat` once (or add to startup).
- Use Chrome normally.
- Before long inactivity/shutdown → run the Save batch (or rely on volatility for OPFS).

## Testing
- Open **https://jchu634.github.io/opfs-test/**
- Add Random files
- Lock workstation (or press Windows key and type logoff and press enter)
- Login to Windows
- Check if OPFS data persists.

## Notes
- After disabling Gemini Nano (via `chrome://flags`), your profile should be much smaller.
- OPFS data is **volatile** by nature — losing it on reboot is usually harmless.
- If issues arise after Chrome updates, re-run the setup batch.

## macOS Version (Cowboy Style)

On macOS you can do a similar targeted OPFS mitigation using a RAM disk:

### Quick Setup (Terminal)
```bash
# 1. Create a 1GB RAM disk
diskutil erasevolume HFS+ "OPFS_RAM" `hdiutil attach -nomount ram://2097152`

# 2. Move existing OPFS (close Chrome first)
mv ~/Library/Application\ Support/Google/Chrome/Default/File\ System /Volumes/OPFS_RAM/OPFS

# 3. Create symlink
ln -s /Volumes/OPFS_RAM/OPFS ~/Library/Application\ Support/Google/Chrome/Default/File\ System
```

### For persistence / automation
- Use `LaunchAgents` or a simple script in `~/.zshrc` / Login items.
- To save: `cp -a /Volumes/OPFS_RAM/OPFS ~/Backups/ChromeOPFS/`

**Note**: macOS `/tmp` is also tmpfs-backed on modern versions and can be used for even simpler cowboy mode.

## Linux Version (Cowboy Style)
- /dev/shm is your friend!

## With hope this helps someone — Fyyre 2026/07/03

# 📁 Rclone + Google Drive Setup on Windows

This guide walks you through installing **rclone**, connecting **one or two Google Drive accounts**, and mounting them as local drives (`X:` and `W:`) using **WinFsp**.

---

## 🔧 Prerequisites

- Windows 10/11 (64-bit recommended)
- Internet access
- Run all scripts **as Administrator**

---

## 🚀 Step 1: Install WinFsp (Required for Mounting)

**WinFsp** enables FUSE file system support on Windows.

### ✅ Install via PowerShell (Admin):

```powershell
# Download official WinFsp installer
Invoke-WebRequest -UseBasicParsing -Uri "https://winfsp.dev/rel/winfsp-install.exe" -OutFile "$env:TEMP\winfsp-install.exe"
# Install silently
Start-Process -Wait -FilePath "$env:TEMP\winfsp-install.exe" -ArgumentList "/silent"
```

> 🔄 **Reboot your PC once** after first installation.

Verify install:
```cmd
sc query WinFsp
```

---

## 📦 Step 2: Install & Configure Rclone

### 📝 `setup_rclone.bat` — First-time install and config for **Account 1**

```batch
@echo off
:: --- CONFIGURATION ---
SET "RCLONE_DIR=C:\rclone"
SET "REMOTE_NAME=gdrive"      :: Remote name for Account 1
SET "MOUNT_LETTER=X:"
SET "CACHE_DIR=C:\rclone_cache"

:: 1. Create dirs
if not exist "%RCLONE_DIR%" mkdir "%RCLONE_DIR%"
if not exist "%CACHE_DIR%" mkdir "%CACHE_DIR%"

:: 2. Install rclone if missing
if not exist "%RCLONE_DIR%\rclone.exe" (
    echo [!] Downloading rclone...
    powershell -Command "Invoke-WebRequest -UseBasicParsing -Uri 'https://downloads.rclone.org/rclone-current-windows-amd64.zip' -OutFile 'rclone.zip'"
    powershell -Command "Expand-Archive -Path 'rclone.zip' -DestinationPath '%RCLONE_DIR%' -Force"
    for /f "delims=" %%i in ('dir /s /b "%RCLONE_DIR%\rclone.exe"') do move "%%i" "%RCLONE_DIR%\rclone.exe" >nul
    del rclone.zip
    echo [+] Rclone installed.
)

:: 3. Run config if no config exists
if not exist "%AppData%\rclone\rclone.conf" (
    echo [!] Configure Google Drive (Account 1)...
    "%RCLONE_DIR%\rclone.exe" config
)

echo [OK] Setup complete for Account 1 (remote: gdrive)
pause
```

> ✅ Run this → follow prompts → **log in with your first Google account**.

---

## ➕ Step 3: Add a Second Google Account

### 📝 Add **Account 2** via command line:

```cmd
rclone config
```

- Choose `n` → name: **`gdrive2`**
- Type: **`drive`**
- Use **default client ID/secret**
- Accept defaults → **auto-config = y**
- **Log in with your SECOND Google account**
- Save and quit (`q`)

> 🔍 Confirm with: `rclone listremotes`

---

## 🗂️ Step 4: Mount Scripts

### 🔹 Mount Google Drive 1 → `X:`  
**File: `mount_gdrive1.bat`**

```batch
@echo off
SET "RCLONE_DIR=C:\rclone"
SET "REMOTE_NAME=gdrive"
SET "MOUNT_LETTER=X:"
SET "CACHE_DIR=C:\rclone_cache"

if not exist "%CACHE_DIR%" mkdir "%CACHE_DIR%"

vol %MOUNT_LETTER% >nul 2>&1 && (echo [!] %MOUNT_LETTER% in use & pause & exit /b 1)

echo [+] Mounting %REMOTE_NAME% to %MOUNT_LETTER%...
start "Rclone1" /min "%RCLONE_DIR%\rclone.exe" mount "%REMOTE_NAME%:" "%MOUNT_LETTER%" ^
    --vfs-cache-mode full ^
    --vfs-cache-max-age 24h ^
    --vfs-cache-max-size 10G ^
    --cache-dir "%CACHE_DIR%" ^
    --no-console

timeout /t 4 >nul
vol %MOUNT_LETTER% >nul && echo [OK] Mounted %MOUNT_LETTER% || echo [!] Mount may have failed.
pause
```

### 🔹 Mount Google Drive 2 → `W:`  
**File: `mount_gdrive2.bat`**

```batch
@echo off
SET "RCLONE_DIR=C:\rclone"
SET "REMOTE_NAME=gdrive2"
SET "MOUNT_LETTER=W:"
SET "CACHE_DIR=C:\rclone_cache2"

if not exist "%CACHE_DIR%" mkdir "%CACHE_DIR%"

vol %MOUNT_LETTER% >nul 2>&1 && (echo [!] %MOUNT_LETTER% in use & pause & exit /b 1)

echo [+] Mounting %REMOTE_NAME% to %MOUNT_LETTER%...
start "Rclone2" /min "%RCLONE_DIR%\rclone.exe" mount "%REMOTE_NAME%:" "%MOUNT_LETTER%" ^
    --vfs-cache-mode full ^
    --vfs-cache-max-age 24h ^
    --vfs-cache-max-size 10G ^
    --cache-dir "%CACHE_DIR%" ^
    --no-console

timeout /t 4 >nul
vol %MOUNT_LETTER% >nul && echo [OK] Mounted %MOUNT_LETTER% || echo [!] Mount may have failed.
pause
```

> ⚠️ **Always run mount scripts as Administrator.**  
> 🗂️ Each remote uses its **own cache directory** to avoid conflicts.

---

## 🔄 Optional: Auto-Mount at Login

Use **Windows Task Scheduler**:
- Trigger: At user logon
- Action: Start `mount_gdrive1.bat` (and `mount_gdrive2.bat`)
- Run with highest privileges

---

## ✅ Verify

After mounting:
- Open **File Explorer** → check `X:\` and `W:\`
- Or run:
  ```cmd
  rclone lsd gdrive:
  rclone lsd gdrive2:
  ```

---

> ℹ️ **Official Links**  
> - Rclone: https://rclone.org  
> - WinFsp: https://winfsp.dev/rel/  

📁 You now have **two Google Drives** mounted as local drives on Windows!

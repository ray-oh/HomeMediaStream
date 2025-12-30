# 🗂️ Managing Multiple Google Accounts for Extra Storage

Using multiple Google accounts is a practical way to gain additional storage, as each new account includes its own **15 GB of free space** shared across **Google Drive, Gmail, and Google Photos**.

---

## 1. 🔑 Key Management Methods

To avoid constantly logging in and out of accounts, use these official and third-party tools:

### **Google Drive for Desktop (Official)**
- Sign into **multiple accounts simultaneously**.
- Creates **separate virtual drives** on your computer (e.g., `G:\` for personal, `H:\` for work).
- Enables seamless file access and management without browser switching.

### **Chrome Browser Profiles**
- Create **dedicated Chrome profiles** for each Google account.
- Each profile maintains its own:
  - Bookmarks
  - Extensions
  - Logged-in sessions (including Drive)
- Prevents accidental data mixing and simplifies tab organization.

### **Third-Party Cloud Managers**
- Tools like **[MultCloud](https://www.multcloud.com/)** or **[Shift](https://shift.com/)** act as **cloud aggregators**.
- View, copy, and move files **between multiple Google Drive accounts** in a single interface.
- No need to switch accounts or download/upload manually.

---

## 2. 🔗 Consolidating Access via Sharing

Instead of managing multiple interfaces, consolidate access into one **"Primary" account**:

### **Folder Sharing**
- In your **secondary account**, create a folder.
- **Share it** with your primary account email.
- Grant **"Editor"** permissions.
- The folder appears in your primary Drive under **"Shared with me"** → **"Add shortcut to Drive"** for direct access.

### **Transfer Ownership**
- Move large files to a secondary account.
- Use **"Transfer ownership"** (via right-click > Share > Advanced) to assign the file to the secondary account.
- Frees up space in your primary account permanently.

> 💡 *Note: Ownership transfer only works between accounts in the same domain or consumer accounts.*

---

## 3. 👨‍👩‍👧‍👦 Google One Family Sharing

If you subscribe to **Google One** (paid storage), you can **share your storage pool** with up to **5 other people**:

### How It Works:
- Create a **Family Group** in your Google Account settings.
- **Usage order**: Each member first uses their **own 15 GB free quota**; only after that is exhausted do they draw from the **shared Google One pool**.
- **Privacy preserved**: Family members **cannot see your files** unless you explicitly share them.

> ✅ Ideal for households: One payment, multiple accounts, private data.

---

## ⚠️ Important Considerations

| Issue | Details |
|------|--------|
| **Terms of Service** | Google doesn’t limit account count, but **creating accounts solely to circumvent storage limits** may violate their [Acceptable Use Policy](https://support.google.com/accounts/answer/1347941). |
| **Inactive Over-Quota Accounts** | If an account stays **over its storage limit for 2+ years**, Google may **delete its content**. |
| **File Size Limit** | **No single file larger than 15 GB** can be uploaded to a **free** Google account — even if you have multiple accounts. (Paid accounts support up to 5 TB per file.) |

---

> 💡 **Best Practice**: Use multiple accounts responsibly — for organization (e.g., personal vs. work) or family sharing — rather than as a loophole. Always back up critical data elsewhere.


# 📁 Rclone + Google Drive Setup on Windows

This guide walks you through installing **rclone**, connecting **one or two Google Drive accounts**, and mounting them as local drives (`X:` and `W:`) using **WinFsp**.

## why would you use rclone instead of google drive sync for mounting g drive locally on laptop
While Google Drive for Desktop is sufficient for basic users, power users often prefer rclone for mounting drives due to its superior control, efficiency, and advanced feature set. 
### Key reasons to use rclone over the official sync client include:
- Transparency and Encryption: Rclone provides built-in, client-side cryptographic encryption, ensuring your files are encrypted before they leave your device. Google's native tools do not offer this feature.
- Linux Compatibility: Google does not offer an official Drive for Desktop client for Linux. Rclone is the standard solution for Linux users to mount Google Drive as a local filesystem.
- Granular Performance Tuning: You can manually configure cache settings (VFS), bandwidth limits, and parallel transfer threads. This is particularly useful for streaming media (e.g., Plex) or handling large libraries without freezing.
- Lightweight and Headless Support: Rclone is a standalone binary that consumes fewer system resources than the "bloated" background processes of official sync apps. It supports "Service Accounts" for unattended operation on servers or build machines where no user is logged in.
- Multi-Cloud Integration: Rclone acts as a single interface for over 40 storage providers (Google Drive, S3, Dropbox, etc.), allowing you to mount or sync between different services without needing multiple apps.
- Advanced Scripting and Automation: Because it is command-line based, rclone is easily integrated into automated backup scripts or scheduled tasks. It also provides detailed logs and checksum verification to confirm successful transfers.
- Bypassing API Quotas: Users can create their own Google API Client ID, which can improve reliability and speed by avoiding the shared traffic limits used by the default rclone or Google clients.

If you still want to use Google Drive - [Intro to Google Drive Sync](https://youtu.be/wjDMPq4Ih3o?si=Ayf2ovn_OlTB1HI_)

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

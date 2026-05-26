# Windows Administration Patterns

## Running Detached Background Processes
- `nohup` from Git Bash does NOT reliably persist on Windows
- Use `cmd.exe /c "start \"Title\" cmd.exe /c \"script.bat > log.txt 2>&1\""` to launch truly detached processes
- The outer `start` creates a new window; the inner `cmd /c` runs the script
- Output must be redirected in the inner command since the window may not be visible

## Killing Stubborn Processes
- Some processes (e.g., Ollama) run elevated and resist normal `taskkill`
- From PowerShell: `Start-Process powershell -ArgumentList '-Command','Stop-Process -Id <PID> -Force' -Verb RunAs`
- This triggers a UAC prompt on the desktop — user must click Yes
- From Git Bash: `taskkill //f //pid <PID>` (double slashes for flag escaping)

## PowerShell Output Recovery
- PowerShell does NOT log console output by default
- **PSReadLine history** (`%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`): commands only, no output, enabled by default
- **Windows Event Log** (Script Block Logging, Event ID 4104): script text only, no output, partially auto-enabled for suspicious blocks
- **Terminal scrollback**: in-memory only, lost when window closes
- **Transcript logging** (`Start-Transcript`): captures commands + full output, must be explicitly enabled

### Setting Up Auto-Transcript (recommended)
Add to PowerShell profile (`$PROFILE`):
```powershell
$transcriptDir = "$HOME\Documents\PowerShell\Transcripts"
if (-not (Test-Path $transcriptDir)) { New-Item -ItemType Directory -Path $transcriptDir -Force | Out-Null }
Start-Transcript -Path "$transcriptDir\PS_$(Get-Date -Format 'yyyyMMdd_HHmmss')_$PID.txt" -Append | Out-Null
```

Profile locations:
- PS 5.1: `~\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1`
- PS 7+: `~\Documents\PowerShell\Microsoft.PowerShell_profile.ps1`

## Google Drive for Desktop: False "symlink" Upload Errors
- Google Drive reports unreadable files as `.symlink` type — it's a catch-all, not accurate detection
- Common causes:
  - **Actual NTFS symlinks** in git repos (Terraform often symlinks shared configs across environments)
  - **VS Live Unit Testing (LUT) cache** (`source/lut/`) — creates shadow copies using file system virtualization; if VS is removed, files become unreadable ghost placeholders
  - **Dehydrated/placeholder files** from any cloud or virtualization provider that's no longer running
- Fix actual symlinks: replace with copies of targets (`cp --remove-destination <target> <symlink>`)
- Fix virtualized ghosts: delete the stale cache directory (e.g., `source/lut/`)
- Google Drive logs: `%LOCALAPPDATA%\Google\DriveFS\Logs\drive_fs_*.txt` — search for `CreateFileW failed` or `file system virtualization` to find the real paths

## Dev Tool Setup (winget)

### winget Package Names (Verified May 2026)
- **Node.js LTS:** `OpenJS.NodeJS.LTS`
- **ripgrep:** `BurntSushi.ripgrep.MSVC` (use MSVC, not GNU — no extra dependencies)
- **OpenCode:** Not on winget. Install via npm: `npm install -g opencode-ai`

### PowerShell Execution Policy Blocks npm
Fresh Windows installs often have `Restricted` execution policy, which blocks `npm.cmd` and other `.cmd` scripts.

Fix (user-scoped, acceptable on locked-down systems):
```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

If locked by group policy, bypass per-session:
```powershell
powershell -ExecutionPolicy Bypass -Command "npm install -g opencode-ai"
```

### PATH After winget Install
winget updates PATH but the current terminal session won't see it. Always restart the terminal after installing Node.js or other PATH-dependent tools.

## Git Bash on Windows Quirks
- `C:\Python38\python.exe` must be written as `/c/Python38/python.exe` in Git Bash
- Use `cmd.exe /c` to run bat files or Windows-native commands
- Backslash paths in quoted strings need care — prefer forward slashes where possible
- `taskkill` flags use `//` instead of `/` in Git Bash (e.g., `taskkill //f //im process.exe`)

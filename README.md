# Windows Disk Space Investigation

## Overview

This document captures findings and remediation steps from a disk space investigation on a Windows Server running SQL Server 2019.

---

## Investigation Method

The following PowerShell command was used to identify the top 20 largest files on the `C:\` drive:

```powershell
Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue |
  Sort-Object Length -Descending |
  Select-Object -First 20 FullName, Length
```

To get a folder-level size summary (more useful for targeting cleanup):

```powershell
Get-ChildItem -Path C:\ -Directory -ErrorAction SilentlyContinue | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
    [PSCustomObject]@{ Folder = $_.FullName; SizeGB = [math]::Round($size / 1GB, 2) }
} | Sort-Object SizeGB -Descending | Select-Object -First 15
```

---

## Findings & Space Reclamation

### 1. SQL Server Patch & Install Files — `C:\Install\`

| File | Notes |
|------|-------|
| `C:\Install\Patching\patching_11_02_2026\SQLServer2019-KB5068404-x64.exe` | Latest SQL patch installer |
| `C:\Install\Service Packs\SQLServer2019-KB5046860-x64.exe` | Old service pack |
| `C:\Install\Service Packs\sqlserver2019-kb5046365-x64_...exe` | Old service pack (duplicate) |
| `C:\Install\Installs\SQL2019\x64\Setup\sql_polybase_java_inst.msi` | SQL 2019 install file |
| `C:\Install\Installs\SQL2019\...\SemanticLanguageDatabase.msi` | SQL 2019 install file (x2) |

#### How to Reclaim Space

**Step 1 — Check current SQL Server patch level:**
```powershell
# Confirm SQL Server is already on the latest patch before deleting anything
Invoke-Sqlcmd -Query "SELECT @@VERSION" -ServerInstance "localhost"
```

**Step 2 — Check the size of the Install folder:**
```powershell
$size = (Get-ChildItem "C:\Install" -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
"Install folder size: {0:N2} GB" -f ($size / 1GB)
```

**Step 3 — Archive to a network share (recommended over deleting):**
```powershell
# Replace \\server\share with your actual network path
robocopy "C:\Install" "\\server\share\Backups\Install" /E /MOVE /LOG:C:\Temp\install_move.log
```

**Step 4 — Or delete if archiving is not needed:**
```powershell
Remove-Item "C:\Install" -Recurse -Force
```

> ⚠️ Coordinate with the DBA team before removing any SQL-related files. Confirm patches are applied and SQL Server is healthy first.

---

### 2. Windows Update Cache — `C:\Windows\SoftwareDistribution\`

| File | Notes |
|------|-------|
| `C:\Windows\SoftwareDistribution\Download\...\Windows10.0-KB5126651-x64.cab` | Downloaded Windows update |

#### How to Reclaim Space

**Step 1 — Check the size:**
```powershell
$size = (Get-ChildItem "C:\Windows\SoftwareDistribution\Download" -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
"SoftwareDistribution Download size: {0:N2} GB" -f ($size / 1GB)
```

**Step 2 — Confirm pending updates are installed:**
```powershell
# Check for any pending updates before clearing
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 5
```

**Step 3 — Clear the download cache:**
```powershell
Stop-Service wuauserv -Force
Stop-Service bits -Force
Remove-Item "C:\Windows\SoftwareDistribution\Download\*" -Recurse -Force
Start-Service wuauserv
Start-Service bits
```

> ✅ Safe to run. Windows will re-download updates if needed. Does not affect installed updates.

---

### 3. Browser DLLs — Edge & Chrome

| File | Notes |
|------|-------|
| `C:\Program Files (x86)\Microsoft\EdgeCore\Optimized\msedge.dll` | Edge core DLL |
| `C:\Program Files (x86)\Microsoft\EdgeWebView\Application\155.0.4283.62\msedge.dll` | EdgeWebView DLL |
| `C:\Program Files (x86)\Microsoft\Edge\Application\155.0.4283.62\msedge.dll` | Edge browser DLL |
| `C:\Program Files (x86)\Microsoft\EdgeCore\155.0.4283.62\msedge.dll` | Edge versioned DLL |
| `C:\Program Files\Google\Chrome\Application\156.0.8078.35\chrome.dll` | Chrome browser DLL |

#### How to Reclaim Space

**Option A — Let browsers self-clean (recommended):**

Both Edge and Chrome automatically remove old versioned folders after a successful update. Ensure browsers are up to date and restart them to trigger cleanup.

**Option B — Check total browser footprint:**
```powershell
@("C:\Program Files (x86)\Microsoft", "C:\Program Files\Google\Chrome") | ForEach-Object {
    $size = (Get-ChildItem $_ -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
    [PSCustomObject]@{ Path = $_; SizeGB = [math]::Round($size / 1GB, 2) }
}
```

**Option C — Uninstall browsers not in use:**

If Edge or Chrome is not required on this server, uninstall via:
```powershell
# Uninstall Chrome silently
$chrome = Get-WmiObject -Class Win32_Product | Where-Object { $_.Name -like "*Chrome*" }
$chrome.Uninstall()
```

> ⚠️ Do not manually delete versioned DLL folders — this can break the browser or dependent applications (e.g. apps using EdgeWebView).

---

### 4. Windows Event Logs — `C:\Windows\System32\winevt\Logs\`

| File | Notes |
|------|-------|
| `C:\Windows\System32\winevt\Logs\Microsoft-Windows-SystemDataArchiver%4Diagnostic.evtx` | Diagnostic event log |

#### How to Reclaim Space

**Step 1 — Check sizes of all event logs:**
```powershell
Get-ChildItem "C:\Windows\System32\winevt\Logs\*.evtx" |
  Sort-Object Length -Descending |
  Select-Object Name, @{N="SizeMB";E={[math]::Round($_.Length/1MB,2)}} |
  Select-Object -First 10
```

**Step 2 — Clear a specific log:**
```powershell
wevtutil cl "Microsoft-Windows-SystemDataArchiver/Diagnostic"
```

**Step 3 — Clear all non-critical diagnostic logs at once:**
```powershell
$logsToSkip = @("System","Application","Security")
Get-WinEvent -ListLog * -ErrorAction SilentlyContinue |
  Where-Object { $_.LogName -notin $logsToSkip -and $_.RecordCount -gt 0 } |
  ForEach-Object {
    wevtutil cl $_.LogName
    Write-Host "Cleared: $($_.LogName)"
  }
```

> ⚠️ Do NOT clear `System`, `Application`, or `Security` logs without exporting them first, especially on a production server.

**Export before clearing (recommended):**
```powershell
wevtutil epl "Microsoft-Windows-SystemDataArchiver/Diagnostic" "C:\Temp\diagnostic_backup.evtx"
wevtutil cl "Microsoft-Windows-SystemDataArchiver/Diagnostic"
```

---

### 5. User Downloads — `C:\Users\<name>\Downloads\`

| File | Notes |
|------|-------|
| `C:\Users\tashvir.babulal\Downloads\Microsoft.DataTools.IntegrationServices.exe` | SSIS installer, likely no longer needed |

#### How to Reclaim Space

**Step 1 — Check size of all user Download folders:**
```powershell
Get-ChildItem "C:\Users" -Directory | ForEach-Object {
    $path = "$($_.FullName)\Downloads"
    if (Test-Path $path) {
        $size = (Get-ChildItem $path -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
        [PSCustomObject]@{ User = $_.Name; SizeGB = [math]::Round($size / 1GB, 2) }
    }
} | Sort-Object SizeGB -Descending
```

**Step 2 — Review and delete old installers manually or via script:**
```powershell
# List files older than 90 days in Downloads
Get-ChildItem "C:\Users\tashvir.babulal\Downloads" |
  Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-90) } |
  Select-Object Name, LastWriteTime, @{N="SizeMB";E={[math]::Round($_.Length/1MB,2)}}
```

> ✅ Safe to delete old installers once the software is confirmed installed and working.

---

### 6. WinSxS & Windows Servicing — `C:\Windows\WinSxS\`

| File | Notes |
|------|-------|
| `C:\Windows\WinSxS\ManifestCache\..._blobs.bin` | Component store manifest cache |
| `C:\Windows\servicing\Sessions\Sessions.xml` | Servicing session data |

#### How to Reclaim Space

WinSxS cannot be deleted manually but can be safely reduced using built-in tools.

**Check WinSxS size:**
```powershell
Dism /Online /Cleanup-Image /AnalyzeComponentStore
```

**Clean up superseded components (safe, recommended):**
```powershell
Dism /Online /Cleanup-Image /StartComponentCleanup
```

**More aggressive cleanup — remove components needed for uninstall of old updates:**
```powershell
Dism /Online /Cleanup-Image /StartComponentCleanup /ResetBase
```

> ⚠️ `/ResetBase` is irreversible — you will no longer be able to uninstall previously installed Windows updates. Only use on stable, production-confirmed systems.

---

### 7. SQL Server Database & Log Files (Additional Check)

SQL Server itself can accumulate space through data and log file growth.

**Check SQL data and log file sizes:**
```powershell
Invoke-Sqlcmd -Query "
  SELECT name, physical_name,
    CAST(size * 8.0 / 1024 / 1024 AS DECIMAL(10,2)) AS SizeGB,
    type_desc
  FROM sys.master_files
  ORDER BY size DESC
" -ServerInstance "localhost"
```

**Shrink transaction log files (use with caution):**
```sql
-- Run in SSMS for the target database
USE [YourDatabaseName];
DBCC SHRINKFILE (YourDatabaseName_log, 1);
```

**Check for databases in FULL recovery with no log backups (log bloat risk):**
```sql
SELECT name, log_reuse_wait_desc, recovery_model_desc
FROM sys.databases
WHERE log_reuse_wait_desc <> 'NOTHING';
```

> ⚠️ Shrinking log files is a short-term fix. The root cause is usually missing log backups or incorrect recovery model. Coordinate with the DBA team.

---

## Full Cleanup Runbook (Priority Order)

| Priority | Action | Estimated Space | Risk |
|----------|--------|----------------|------|
| 1 | Archive/delete `C:\Install\` | High (several GB) | Low — confirm SQL patched first |
| 2 | Clear `SoftwareDistribution\Download` | Medium | None |
| 3 | DISM component store cleanup | Medium | Low |
| 4 | Clear diagnostic event logs | Low–Medium | Low — export first |
| 5 | Clean user Downloads folders | Low–Medium | None |
| 6 | Uninstall unused browsers | Low | Low |
| 7 | SQL log file review | Varies | Medium — involve DBA |

---

## Quick One-Shot Disk Cleanup (Safe)

Run this to handle temp files, update remnants, and recycle bin in one go:

```powershell
# Windows Disk Cleanup with all options selected
cleanmgr /sageset:1
cleanmgr /sagerun:1
```

Or silently via DISM + SoftwareDistribution clear:

```powershell
Stop-Service wuauserv -Force
Stop-Service bits -Force
Remove-Item "C:\Windows\SoftwareDistribution\Download\*" -Recurse -Force
Start-Service wuauserv
Start-Service bits
Dism /Online /Cleanup-Image /StartComponentCleanup
Remove-Item "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue
Write-Host "Cleanup complete."
```

---

## Notes

- Investigation performed on a Windows Server with SQL Server 2019 installed
- Always verify patches are applied before deleting install/patch files
- Coordinate with the DBA team before removing any SQL-related files
- Export event logs before clearing on production servers
- Run cleanup steps during a maintenance window where possible

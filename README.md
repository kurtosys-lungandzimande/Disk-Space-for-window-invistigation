# Windows Disk Space Investigation

## Overview

This document captures findings from a disk space investigation on a Windows Server running SQL Server 2019. The C:\ drive is critically low at only 3.24 GB free (4%). This is investigation only — no cleanup actions have been taken yet.

---

## Drive Status Summary

| Drive | Used (GB) | Free (GB) | Total (GB) | Status |
|-------|-----------|-----------|------------|--------|
| C:\   | 76.75     | 3.24      | 80         | 🔴 CRITICAL — 4% free |
| H:\   | 231.47    | 68.51     | 299.98     | 🟡 Monitor |
| L:\   | 289.40    | 110.60    | 400        | 🟢 OK |
| M:\   | 336.82    | 263.18    | 600        | 🟢 OK |
| T:\   | 6.02      | 193.97    | 200        | 🟢 OK |

---

## Investigation Commands Used

**Top 20 largest files:**
```powershell
Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue |
  Sort-Object Length -Descending |
  Select-Object -First 20 FullName, Length
```

**Top-level folder sizes:**
```powershell
Get-ChildItem -Path C:\ -Directory -ErrorAction SilentlyContinue | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
    [PSCustomObject]@{ Folder = $_.FullName; SizeGB = [math]::Round($size / 1GB, 2) }
} | Sort-Object SizeGB -Descending | Select-Object -First 15
```

**Windows subfolder sizes:**
```powershell
Get-ChildItem -Path C:\Windows -Directory -ErrorAction SilentlyContinue | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
    [PSCustomObject]@{ Folder = $_.FullName; SizeGB = [math]::Round($size / 1GB, 2) }
} | Sort-Object SizeGB -Descending | Select-Object -First 10
```

**Octopus Deploy folder sizes:**
```powershell
@("C:\Octopus\Packages", "C:\Octopus\TaskLogs", "C:\Octopus\Artifacts", "C:\Octopus\Files") | ForEach-Object {
    if (Test-Path $_) {
        $size = (Get-ChildItem $_ -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
        [PSCustomObject]@{ Folder = $_; SizeGB = [math]::Round($size / 1GB, 2) }
    }
}
```

**User Downloads sizes:**
```powershell
Get-ChildItem "C:\Users" -Directory | ForEach-Object {
    $path = "$($_.FullName)\Downloads"
    if (Test-Path $path) {
        $size = (Get-ChildItem $path -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
        [PSCustomObject]@{ User = $_.Name; SizeGB = [math]::Round($size / 1GB, 2) }
    }
} | Sort-Object SizeGB -Descending
```

**Event log sizes:**
```powershell
Get-ChildItem "C:\Windows\System32\winevt\Logs\*.evtx" |
  Sort-Object Length -Descending |
  Select-Object Name, @{N="SizeMB";E={[math]::Round($_.Length/1MB,2)}} |
  Select-Object -First 10
```

---

## C:\ Top-Level Folder Breakdown (Actual)

| Folder | Size (GB) | Notes |
|--------|-----------|-------|
| `C:\Windows` | 30.23 | OS files — partially cleanable |
| `C:\Program Files` | 14.49 | Installed apps |
| `C:\Program Files (x86)` | 7.15 | 32-bit installed apps |
| `C:\Install` | 4.18 | SQL patch/install files — recoverable |
| `C:\Octopus` | 3.90 | Octopus Deploy — partially recoverable |
| `C:\Users` | 0.88 | User profiles |
| `C:\opscode` | 0.71 | Chef client |
| `C:\chef` | 0.13 | Chef config |
| `C:\tools` | 0.04 | Tools |
| `C:\var` | 0.04 | Misc |

---

## C:\Windows Subfolder Breakdown (Actual)

| Folder | Size (GB) | Recoverable | Notes |
|--------|-----------|-------------|-------|
| `WinSxS` | 10.44 | ~3–5 GB | DISM cleanup |
| `SoftwareDistribution` | 4.99 | ~4.99 GB | Windows Update cache — fully recoverable |
| `assembly` | 4.83 | ❌ | .NET GAC — do not touch |
| `System32` | 4.24 | ❌ | System files — do not touch |
| `servicing` | 2.44 | ~1–2 GB | DISM cleanup |
| `SysWOW64` | 1.08 | ❌ | System files — do not touch |
| `Microsoft.NET` | 0.69 | ❌ | .NET runtime — do not touch |
| `Fonts` | 0.36 | ❌ | System fonts |

---

## Octopus Deploy — `C:\Octopus\Files\` (Actual)

Only `C:\Octopus\Files` exists with **0.72 GB** of old versioned deployment package zips:

| File | Size (MB) |
|------|-----------|
| `ksys423_loaders@S1.0.3+...zip` | 76.35 |
| `ksys437_loaders@S1.0.22+-DEV...zip` | 70.03 |
| `ksys437_loaders@S1.0.21+-DEV...zip` | 70.03 |
| `ksys437_loaders@S1.0.20+-DEV...zip` | 70.03 |
| `ksys463_web_loaders@S1.0.4+-DEV...zip` | 69.84 |
| `ksys373_loaders@S1.0.7+...zip` | 66.60 |
| `ksys177_loaders@S1.0.3+-DEV...zip` | 60.95 |
| `ksys177_loaders@S1.0.2+-STAGING...zip` | 60.94 |
| `ksys235_loaders@S1.0.2+-STAGING...zip` | 55.64 |
| `ksys235_loaders@S1.0.3+-DEV...zip` | 55.60 |

> Multiple old DEV and STAGING versioned packages present. Octopus retention policy should manage these.

---

## User Downloads (Actual)

| User | Size (GB) | Notes |
|------|-----------|-------|
| tashvir.babulal | 0.52 | Contains `Microsoft.DataTools.IntegrationServices.exe` |
| divashan.naicker | 0.03 | Low |
| bryan.bamber | 0.01 | Low |
| All others | 0 | Negligible |

---

## Event Log Sizes (Actual)

| Log File | Size (MB) | Action |
|----------|-----------|--------|
| `Microsoft-Windows-SystemDataArchiver%4Diagnostic.evtx` | 300 | 🔴 Unusually large — export then clear |
| `Microsoft-Windows-Storage-Storport%4Operational.evtx` | 32 | Low priority |
| `Microsoft-Windows-Ntfs%4Operational.evtx` | 32 | Low priority |
| `Application.evtx` | 20 | ❌ Do not clear |
| `System.evtx` | 20 | ❌ Do not clear |
| `Security.evtx` | 20 | ❌ Do not clear |
| `Microsoft-Windows-LAPS%4Operational.evtx` | 16.07 | Low priority |
| `Microsoft-Windows-Store%4Operational.evtx` | 16.07 | Low priority |
| `Microsoft-Windows-Windows Defender%4Operational.evtx` | 16 | Low priority |
| `Microsoft-Windows-Partition%4Diagnostic.evtx` | 15.07 | Low priority |

---

## Space Reclamation Summary

| # | Area | Actual Size | Estimated Recoverable | Risk |
|---|------|------------|----------------------|------|
| 1 | `C:\Windows\SoftwareDistribution` | 4.99 GB | ~4.99 GB | None |
| 2 | `C:\Install\` | 4.18 GB | ~4.18 GB | Low — confirm SQL patched first |
| 3 | `C:\Windows\WinSxS` + `servicing` | 12.88 GB | ~3–5 GB | Low |
| 4 | `C:\Octopus\Files` | 0.72 GB | ~0.50 GB | Low — use Octopus retention policy |
| 5 | Event logs (diagnostic only) | ~0.36 GB | ~0.30 GB | Low — export first |
| 6 | User Downloads | 0.52 GB | ~0.52 GB | None |
| 7 | `C:\Windows\Temp` + `%TEMP%` | ~0.5 GB est. | ~0.50 GB | None |
| | **Total Estimated Recoverable** | | **~14–16 GB** | |

> Reclaiming 14–16 GB would bring C:\ free space from 3.24 GB to approximately 17–19 GB (~21% free).

---

## Findings & Space Reclamation Steps

### 1. SQL Server Patch & Install Files — `C:\Install\`

| File | Notes |
|------|-------|
| `C:\Install\Patching\patching_11_02_2026\SQLServer2019-KB5068404-x64.exe` | Latest SQL patch installer |
| `C:\Install\Service Packs\SQLServer2019-KB5046860-x64.exe` | Old service pack |
| `C:\Install\Service Packs\sqlserver2019-kb5046365-x64_...exe` | Old service pack (duplicate) |
| `C:\Install\Installs\SQL2019\x64\Setup\sql_polybase_java_inst.msi` | SQL 2019 install file |
| `C:\Install\Installs\SQL2019\...\SemanticLanguageDatabase.msi` | SQL 2019 install file (x2) |

**Step 1 — Confirm SQL Server patch level:**
```powershell
Invoke-Sqlcmd -Query "SELECT @@VERSION" -ServerInstance "localhost"
```

**Step 2 — Archive to network share (recommended):**
```powershell
robocopy "C:\Install" "\\server\share\Backups\Install" /E /MOVE /LOG:C:\Temp\install_move.log
```

**Step 3 — Or delete if archiving not needed:**
```powershell
Remove-Item "C:\Install" -Recurse -Force
```

> ⚠️ Coordinate with the DBA team. Confirm SQL Server is healthy and fully patched before removing.

---

### 2. Windows Update Cache — `C:\Windows\SoftwareDistribution\`

**Clear the download cache:**
```powershell
Stop-Service wuauserv -Force
Stop-Service bits -Force
Remove-Item "C:\Windows\SoftwareDistribution\Download\*" -Recurse -Force
Start-Service wuauserv
Start-Service bits
```

> ✅ Safe. Windows re-downloads updates if needed. Does not affect already installed updates.

---

### 3. WinSxS & Windows Servicing — `C:\Windows\WinSxS\`

**Analyse first:**
```powershell
Dism /Online /Cleanup-Image /AnalyzeComponentStore
```

**Safe cleanup:**
```powershell
Dism /Online /Cleanup-Image /StartComponentCleanup
```

**Aggressive cleanup (irreversible):**
```powershell
Dism /Online /Cleanup-Image /StartComponentCleanup /ResetBase
```

> ⚠️ `/ResetBase` prevents uninstalling old Windows updates. Only use on a stable, confirmed production system.

---

### 4. Octopus Deploy — `C:\Octopus\Files\`

**Recommended — use Octopus built-in retention policy:**
1. Log into Octopus Deploy web portal
2. Go to **Configuration > Retention Policies** and set limits per environment/project
3. Or go to **Library > Packages** and configure how many versions to keep

**Check current files:**
```powershell
Get-ChildItem "C:\Octopus\Files" |
  Sort-Object Length -Descending |
  Select-Object Name, @{N="SizeMB";E={[math]::Round($_.Length/1MB,2)}}
```

> ⚠️ Do not manually delete files from `C:\Octopus\Files` — let Octopus manage retention to avoid breaking deployment history.

---

### 5. Windows Event Logs

**Export before clearing:**
```powershell
wevtutil epl "Microsoft-Windows-SystemDataArchiver/Diagnostic" "C:\Temp\diagnostic_backup.evtx"
wevtutil cl "Microsoft-Windows-SystemDataArchiver/Diagnostic"
```

**Check all log sizes:**
```powershell
Get-ChildItem "C:\Windows\System32\winevt\Logs\*.evtx" |
  Sort-Object Length -Descending |
  Select-Object Name, @{N="SizeMB";E={[math]::Round($_.Length/1MB,2)}} |
  Select-Object -First 10
```

> ⚠️ Do NOT clear `System`, `Application`, or `Security` logs on a production server.

---

### 6. User Downloads — `C:\Users\<name>\Downloads\`

**List files older than 90 days:**
```powershell
Get-ChildItem "C:\Users\tashvir.babulal\Downloads" |
  Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-90) } |
  Select-Object Name, LastWriteTime, @{N="SizeMB";E={[math]::Round($_.Length/1MB,2)}}
```

> ✅ Safe to delete old installers once software is confirmed installed and working.

---

### 7. Temp Files

```powershell
Remove-Item "$env:TEMP\*" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "C:\Windows\Temp\*" -Recurse -Force -ErrorAction SilentlyContinue
```

> ✅ Safe. Temp files are recreated as needed.

---

### 8. SQL Server Database & Log Files (Additional Check)

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

**Check for log bloat:**
```sql
SELECT name, log_reuse_wait_desc, recovery_model_desc
FROM sys.databases
WHERE log_reuse_wait_desc <> 'NOTHING';
```

> ⚠️ Involve the DBA team before any SQL file changes.

---

## Full Cleanup Runbook (Priority Order)

| Priority | Action | Estimated Recoverable | Risk |
|----------|--------|-----------------------|------|
| 1 | Clear `SoftwareDistribution\Download` | ~4.99 GB | None |
| 2 | Archive/delete `C:\Install\` | ~4.18 GB | Low — confirm SQL patched first |
| 3 | DISM WinSxS + servicing cleanup | ~3–5 GB | Low |
| 4 | Clean user Downloads | ~0.52 GB | None |
| 5 | Clear `C:\Windows\Temp` + `%TEMP%` | ~0.50 GB | None |
| 6 | Octopus retention policy cleanup | ~0.50 GB | Low |
| 7 | Clear diagnostic event logs | ~0.30 GB | Low — export first |
| 8 | SQL log file review | Varies | Medium — involve DBA |
| | **Total** | **~14–16 GB** | |

---

## Recommendations

Based on the investigation, the following actions are recommended before any cleanup is performed:

1. **Schedule a maintenance window** — C:\ is critically low at 3.24 GB free. This should be treated as urgent.
2. **Involve the DBA team** before touching anything under `C:\Install\` or SQL log files.
3. **Start with zero-risk items first** — SoftwareDistribution, Temp files, and User Downloads require no approvals.
4. **Use built-in tools** (DISM, Octopus retention policy) rather than manual deletion where possible.
5. **Export event logs** before clearing, especially on a production server.
6. **Archive `C:\Install\` to a network share** rather than deleting outright — keeps a safety copy.

---

## Before & After Estimate

| Scenario | Free Space on C:\ | % Free |
|----------|-------------------|--------|
| Current (no action) | 3.24 GB | 4% 🔴 |
| After zero-risk cleanup only (items 1, 6, 7) | ~8.75 GB | ~11% 🟡 |
| After all recommended cleanup (items 1–7) | ~17–19 GB | ~21–24% 🟢 |
| After all cleanup including SQL review | ~17–19 GB+ | ~21%+ 🟢 |

> General best practice is to keep at least 15–20% free on a system drive. The target should be a minimum of 12 GB free on this 80 GB C:\ drive.

---

## Notes

- Investigation performed on a Windows Server with SQL Server 2019 installed
- C:\ is critically low — cleanup should be scheduled in the next maintenance window
- Always verify SQL patches are applied before deleting install/patch files
- Coordinate with the DBA team before removing any SQL-related files
- Export event logs before clearing on production servers
- No cleanup actions have been taken yet — this document is investigation only

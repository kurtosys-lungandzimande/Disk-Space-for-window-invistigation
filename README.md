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

---

## Findings

### 1. SQL Server Patch & Install Files — `C:\Install\`

| File | Notes |
|------|-------|
| `C:\Install\Patching\patching_11_02_2026\SQLServer2019-KB5068404-x64.exe` | Latest SQL patch installer |
| `C:\Install\Service Packs\SQLServer2019-KB5046860-x64.exe` | Old service pack |
| `C:\Install\Service Packs\sqlserver2019-kb5046365-x64_...exe` | Old service pack (duplicate) |
| `C:\Install\Installs\SQL2019\x64\Setup\sql_polybase_java_inst.msi` | SQL 2019 install file |
| `C:\Install\Installs\SQL2019\...\SemanticLanguageDatabase.msi` | SQL 2019 install file (x2) |

**Action:** If SQL Server is already patched and running, the entire `C:\Install\` folder can be archived to a network share or deleted to reclaim significant space.

---

### 2. Windows Update Cache — `C:\Windows\SoftwareDistribution\`

| File | Notes |
|------|-------|
| `C:\Windows\SoftwareDistribution\Download\...\Windows10.0-KB5126651-x64.cab` | Downloaded Windows update |

**Action:** Safe to clear after updates are applied.

```powershell
Stop-Service wuauserv
Remove-Item C:\Windows\SoftwareDistribution\Download\* -Recurse -Force
Start-Service wuauserv
```

---

### 3. Browser DLLs — Edge & Chrome

| File | Notes |
|------|-------|
| `C:\Program Files (x86)\Microsoft\EdgeCore\Optimized\msedge.dll` | Edge core DLL |
| `C:\Program Files (x86)\Microsoft\EdgeWebView\Application\155.0.4283.62\msedge.dll` | EdgeWebView DLL |
| `C:\Program Files (x86)\Microsoft\Edge\Application\155.0.4283.62\msedge.dll` | Edge browser DLL |
| `C:\Program Files (x86)\Microsoft\EdgeCore\155.0.4283.62\msedge.dll` | Edge versioned DLL |
| `C:\Program Files\Google\Chrome\Application\156.0.8078.35\chrome.dll` | Chrome browser DLL |

**Action:** Do not delete manually. Both Edge and Chrome retain old versioned DLLs during updates and self-clean over time. Uninstalling unused browsers is the safest approach if they are not needed.

---

### 4. Windows Event Log

| File | Notes |
|------|-------|
| `C:\Windows\System32\winevt\Logs\Microsoft-Windows-SystemDataArchiver%4Diagnostic.evtx` | Diagnostic event log |

**Action:** Can be cleared if not required for auditing. Use Event Viewer or:

```powershell
wevtutil cl "Microsoft-Windows-SystemDataArchiver/Diagnostic"
```

---

### 5. Other System Files (Do Not Delete)

| File | Notes |
|------|-------|
| `C:\Windows\System32\MRT.exe` | Microsoft Malicious Software Removal Tool — system file |
| `C:\Windows\servicing\Sessions\Sessions.xml` | Windows servicing session data — system file |
| `C:\Windows\WinSxS\ManifestCache\..._blobs.bin` | WinSxS component cache — managed by Windows |
| `C:\Program Files (x86)\AWS Tools\PowerShell\AWSPowerShell\AWSPowerShell.dll-Help.xml` | AWS PowerShell module help file |

**Action:** Leave these files alone. They are managed by the OS or installed tools.

---

## Recommended Cleanup Steps

1. **Archive or delete `C:\Install\`** — highest impact, safe if SQL Server is already patched
2. **Clear Windows Update download cache** — run the PowerShell commands above
3. **Run Windows Disk Cleanup** for temp files, thumbnails, and old update remnants:
   ```powershell
   cleanmgr /sageset:1
   cleanmgr /sagerun:1
   ```
4. **Check Recycle Bin** — empty if not needed
5. **Review `C:\Users\<name>\Downloads`** — e.g. `Microsoft.DataTools.IntegrationServices.exe` found here

---

## Notes

- Investigation performed on a Windows Server with SQL Server 2019 installed
- Always verify patches are applied before deleting install/patch files
- Coordinate with the DBA team before removing any SQL-related files

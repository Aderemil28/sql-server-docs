# Upgrade SSMS 21 / 22 in place (Visual Studio Installer)

**Status:** Tested on Dev · **Applies to:** SSMS 21 and 22 installed through the Visual Studio Installer · **Steps:** 10

Upgrade SQL Server Management Studio to the latest release on a server where it was installed through the Visual Studio Installer, using the same silent command that SCCM or another deployment tool would run. Then confirm the new version and check that the old build left nothing behind for a vulnerability scanner to flag.

This was first run on a Dev server as the manual test the software team asked for before automating SSMS upgrades through SCCM: **22.1.0 to 22.11.1, exit code 0, no reboot**. In this runbook **DEVSQL01** is the server being upgraded. Replace it with your own.

> [!NOTE]
> **Before you start**
>
> - An approved request (on Dev, a simple service request is enough: SSMS is a client tool and the SQL Server engine, databases and services aren't touched)
> - Local administrator on the server, and an **elevated** PowerShell or PowerShell ISE window
> - The server can reach Microsoft's download servers through the proxy, or you have an offline layout (Step 3)
> - Nobody is using SSMS on the server during the upgrade
> - The scripts [Get-SSMSInstallInfo.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Get-SSMSInstallInfo.ps1) and [Find-SSMSLeftoverFiles.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Find-SSMSLeftoverFiles.ps1) saved on the server, for example in `C:\Temp`

> [!IMPORTANT]
> SSMS 21 and later are installed and updated by the **Visual Studio Installer**, not by a standalone setup like SSMS 18, 19 and 20. That's why the old "download the new SSMS and run setup" approach doesn't apply, and why the version doesn't always show where you'd expect.

## Step 1: Open the request

Keep it short. For example:

> Upgrading SSMS 22.1.0 to the latest release on DEVSQL01. This is the manual test run the software team asked for before SSMS upgrades are automated through SCCM. Dev server, client tool only, no impact to SQL Server services.

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-01.png and uncomment the next line -->
<!-- ![Step 1: Open the request](../images/upgrade-ssms-22/step-01.png) -->

> [!TIP]
> **Expected result:** The request is approved and the team knows SSMS on DEVSQL01 will be unavailable for about 30 minutes.

## Step 2: Check the current version and install path

[Get-SSMSInstallInfo.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Get-SSMSInstallInfo.ps1) is read-only unless you add `-Upgrade`. It finds the Visual Studio Installer, lists every SSMS 21+ instance with its version, path and channel, checks the registry for older SSMS 18/19/20 installs, and prints the upgrade command.

In an elevated PowerShell ISE window, put these three lines in a new tab and press **F5**:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force   # this window only; machine policy unchanged
cd C:\Temp
.\Get-SSMSInstallInfo.ps1
```

Two things that catch people out in ISE:

- **Each command on its own line.** If two lines get pasted together (for example `.\Get-SSMSInstallInfo.ps1Set-ExecutionPolicy ...`), PowerShell reads it as one long file name and fails
- **Closing ISE resets the session.** The execution policy bypass is lost and the folder goes back to `C:\Windows\system32`, so re-run all three lines after reopening it

Write down the **version**, **install path** and **channel**. The install path isn't always on C:. On the test server it was `D:\Program Files\Microsoft SQL Server Management Studio 22\Release`.

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-02.png and uncomment the next line -->
<!-- ![Step 2: Check the current version and install path](../images/upgrade-ssms-22/step-02.png) -->

> [!TIP]
> **Expected result:** One SSMS instance listed with its version (for example 22.1.0), install path and channel (Release), and an upgrade command printed at the end.

## Step 3: Check the server can download the update

The update downloads from Microsoft. First, the Visual Studio Installer **updates itself**, then it downloads the new SSMS build. If the proxy blocks either step, the upgrade fails or appears to hang.

```powershell
Test-NetConnection -ComputerName download.visualstudio.microsoft.com -Port 443
Test-NetConnection -ComputerName aka.ms -Port 443
```

If either shows `TcpTestSucceeded : False`, use an **offline layout** instead. Build it on a machine with internet access using the SSMS bootstrapper (`vs_SSMS.exe`, from the SSMS download page), copy the folder to the server, and update from it:

```bat
:: On a machine with internet access
vs_SSMS.exe --layout C:\SSMS_Layout --lang en-US

:: On the server, after copying C:\SSMS_Layout across
cd C:\SSMS_Layout
vs_SSMS.exe update --noWeb --quiet --norestart
```

The SCCM package will need the same decision: either the servers can reach Microsoft, or the package carries an offline layout.

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-03.png and uncomment the next line -->
<!-- ![Step 3: Check the server can download the update](../images/upgrade-ssms-22/step-03.png) -->

> [!TIP]
> **Expected result:** Both tests succeed, or an offline layout is ready on the server.

## Step 4: Close SSMS everywhere on the server

The update fails if SSMS is running in **any** session on the server, not only yours.

```powershell
Get-Process -Name Ssms -IncludeUserName -ErrorAction SilentlyContinue |
    Select-Object Id, UserName, StartTime
```

Ask anyone listed to close it. If they've disconnected and left it open, end it (after checking with them):

```powershell
Stop-Process -Name Ssms -Force
```

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-04.png and uncomment the next line -->
<!-- ![Step 4: Close SSMS everywhere on the server](../images/upgrade-ssms-22/step-04.png) -->

> [!TIP]
> **Expected result:** The command returns nothing: no SSMS process is running.

## Step 5: Run the upgrade

Use the script rather than clicking **Update** in the Visual Studio Installer. The script runs the same silent command a deployment tool would, so this run proves the process SCCM will automate, and it gives you an exit code and the before and after versions for the ticket.

In the ISE tab, change the last line and press **F5**:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
cd C:\Temp
.\Get-SSMSInstallInfo.ps1 -Upgrade
```

The command it runs for each SSMS instance is:

```bat
"C:\Program Files (x86)\Microsoft Visual Studio\Installer\setup.exe" update --installPath "<install path from Step 2>" --quiet --norestart
```

Open the **Visual Studio Installer** (`C:\Program Files (x86)\Microsoft Visual Studio\Installer\setup.exe`, or search "Visual Studio Installer" in the Start menu) and leave it open while the script runs. The console shows nothing until the end, but the installer window shows download and install progress, and any error is clearer there than in the exit code.

Expect **10 to 20 minutes**. On the test server the installer updated itself first (to version 4.11.24.2075), then updated SSMS.

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-05.png and uncomment the next line -->
<!-- ![Step 5: Run the upgrade](../images/upgrade-ssms-22/step-05.png) -->

> [!TIP]
> **Expected result:** `Exit code: 0` and the new version listed (22.11.1 on the test server). See Step 6 for anything else.

## Step 6: If it fails or seems stuck

**Seems stuck.** If the Visual Studio Installer window shows no progress for more than 20 minutes, open Task Manager and check whether `setup.exe` or `vs_installer` is using CPU or network. Idle usually means the download is blocked: go back to Step 3.

**Exit codes** (from Microsoft's SSMS command-line documentation):

| Exit code | Meaning | What to do |
|---|---|---|
| 0 | Success | Carry on to Step 7 |
| 3010 | Success, reboot needed before SSMS can be used | Schedule a reboot |
| 740 | Elevation required | Run ISE or PowerShell as administrator |
| 1001 | The Visual Studio Installer is already running | Close the installer window (or wait for another update to finish), run again |
| 1003 / 8006 | SSMS is in use / SSMS processes running | Step 4 |
| 1618 | Another installation is running | Wait for it to finish (often Windows Update) |
| 5003 / -1073720687 | Download or connectivity failure | Step 3: proxy or offline layout |
| Other (for example 1603) | General failure | Check the logs below |

**Logs** for each run are in the `%TEMP%` folder of the account that ran it. Look for files starting with `ssms_bootstrapper`, `ssms_client` and `ssms_setup`:

```powershell
Get-ChildItem $env:TEMP -Filter 'ssms_*' | Sort-Object LastWriteTime -Descending | Select-Object -First 5 Name, LastWriteTime
```

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-06.png and uncomment the next line -->
<!-- ![Step 6: If it fails or seems stuck](../images/upgrade-ssms-22/step-06.png) -->

> [!TIP]
> **Expected result:** You know the cause from the exit code or log, and the re-run returns 0.

## Step 7: Confirm the new version

1. Run the check again, without `-Upgrade`, and confirm the new version and that the install path hasn't changed (the update installs in place):

   ```powershell
   .\Get-SSMSInstallInfo.ps1
   ```

2. Confirm **Add or Remove Programs** shows the new version too. Most vulnerability scanners read this entry:

   ```powershell
   Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
                    'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' |
       Where-Object DisplayName -like '*Management Studio*' |
       Select-Object DisplayName, DisplayVersion, InstallLocation
   ```

3. Open SSMS, check **Help > About**, and connect to the local instance.

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-07.png and uncomment the next line -->
<!-- ![Step 7: Confirm the new version](../images/upgrade-ssms-22/step-07.png) -->

> [!TIP]
> **Expected result:** The script, Add or Remove Programs and Help > About all show the new version, and SSMS connects.

## Step 8: Check nothing from the old version was left behind

If the upgrade was done to close a vulnerability finding, check before the next scan that no old binaries remain. [Find-SSMSLeftoverFiles.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Find-SSMSLeftoverFiles.ps1) is read-only. It looks in the SSMS install folders and the Visual Studio package cache for files still carrying the old version number, any curl / libcurl DLLs, and old install folders.

Give it the old version prefix, the time the upgrade **started**, and the install path from Step 2 (the default search only covers C:):

```powershell
.\Find-SSMSLeftoverFiles.ps1 -OldVersionPrefix '22.1.' -UpgradeTime '2026-10-22 10:20' `
    -InstallPath 'D:\Program Files\Microsoft SQL Server Management Studio 22'
```

Files written after `-UpgradeTime` are marked **Upgraded**, because they came from the new install. Use the start time, not the finish time, or files written during the upgrade are flagged. That matters because some components bundled with SSMS have their own version numbers that happen to look like the old SSMS build. On the test server, four Analysis Services and BI DLLs showed version **22.1.17.0**, but their timestamps matched the upgrade: they were installed fresh, not left over.

The script writes a CSV report to `C:\Temp` and prints a count by status.

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-08.png and uncomment the next line -->
<!-- ![Step 8: Check nothing from the old version was left behind](../images/upgrade-ssms-22/step-08.png) -->

> [!TIP]
> **Expected result:** *Clean*, or every match is marked **Upgraded** (written during the upgrade). No curl binaries, one SSMS folder, no old payloads in the package cache.

## Step 9: Record the result and hand off to the deployment team

Ticket note:

> SSMS upgraded from 22.1.0 to 22.11.1 on DEVSQL01 using the Visual Studio Installer silent update command. Exit code 0, no reboot required. The Visual Studio Installer updated itself as part of the process. SSMS launched and connected successfully. Leftover-file check clean.

For the SCCM (or other deployment) team, the points their package has to handle:

| Point | Detail |
|---|---|
| Command | `setup.exe update --installPath "<path>" --quiet --norestart`, run elevated |
| Install path | Read it per server (`vswhere.exe`); it isn't always `C:\Program Files` |
| Installer self-update | The Visual Studio Installer updates itself first. Allow it, or use `--noUpdateInstaller` (then the run fails with a non-zero code if an installer update is required) |
| Download | Servers need access to Microsoft through the proxy, or the package carries an offline layout (Step 3) |
| SSMS open | Exit code 1003 / 8006 if SSMS is running in any session |
| Reboot | Not needed on the test run (0). Treat 3010 as success with a reboot pending |
| Detection | New version in `vswhere.exe` output or in Add or Remove Programs |

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-09.png and uncomment the next line -->
<!-- ![Step 9: Record the result and hand off](../images/upgrade-ssms-22/step-09.png) -->

> [!TIP]
> **Expected result:** The ticket is updated and closed, and the deployment team has what it needs to build the package.

## Step 10: Roll back if needed

Microsoft doesn't document a rollback for SSMS 21+: updates and channel switches only move forward. If the new version causes a problem:

1. Open the Visual Studio Installer. If it offers **More > Roll back to previous version** for SSMS, use that
2. Otherwise uninstall SSMS (**Visual Studio Installer > More > Uninstall**), then download the bootstrapper for the previous version from the [SSMS release history](https://learn.microsoft.com/ssms/release-history) and install it to the same path

SSMS is a client tool, so SQL Server keeps running throughout.

<!-- Screenshot: save as docs/images/upgrade-ssms-22/step-10.png and uncomment the next line -->
<!-- ![Step 10: Roll back if needed](../images/upgrade-ssms-22/step-10.png) -->

> [!TIP]
> **Expected result:** The previous SSMS version is installed and connects.

## Validation sign-off

- [ ] Request approved before the upgrade
- [ ] Old version, install path and channel recorded (Step 2)
- [ ] Download access confirmed, or offline layout used
- [ ] No SSMS process running when the upgrade started
- [ ] Exit code 0 (or 3010 with the reboot done)
- [ ] New version shown by the script, Add or Remove Programs and Help > About
- [ ] SSMS connects to the local instance
- [ ] Leftover-file check clean, or every match explained by its timestamp
- [ ] Ticket updated; hand-off points sent to the deployment team

## Sources

- **Microsoft Learn:** [Update SQL Server Management Studio](https://learn.microsoft.com/ssms/install/update)
- **Microsoft Learn:** [SSMS command-line parameters and exit codes](https://learn.microsoft.com/ssms/install/command-line-parameters)
- **Microsoft Learn:** [SSMS command-line examples (update, offline layout)](https://learn.microsoft.com/en-us/ssms/install/command-line-examples)
- **Microsoft Learn:** [SSMS 22 release notes](https://learn.microsoft.com/en-us/ssms/release-notes-22)
- **Microsoft Learn:** [SSMS release history](https://learn.microsoft.com/ssms/release-history)
- **MSSQLTips:** [SQL Server Management Studio 22 download, install and configure](https://www.mssqltips.com/sqlservertip/11548/sql-server-management-studio-22-download/)
- **Toolkit:** [Get-SSMSInstallInfo.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Get-SSMSInstallInfo.ps1), [Find-SSMSLeftoverFiles.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Find-SSMSLeftoverFiles.ps1)

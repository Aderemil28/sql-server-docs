# Patch an availability group or distributed AG in rolling order (SQL CU and Windows)

**Status:** Tested in lab · **Applies to:** SQL Server 2019 / 2022 Always On availability groups, including two AGs linked by a distributed AG · **Steps:** 9

Bring every replica up to the current SQL Server cumulative update and the latest Windows updates without an outage: one node at a time, secondary first, a planned failover, then the old primary, with a health check between every step. Three scripts do the work and stop at the first problem. A status report at the end shows exactly where every AG landed.

This was run end to end on a four-node lab: two AGs linked by a distributed AG. In this runbook the names are:

| Name | Role |
|---|---|
| **AG1** | SQLNODE01 (primary) and SQLNODE02, synchronous commit, automatic failover. Global primary of the DAG |
| **AG2** | SQLNODE03 (primary, the DAG forwarder) and SQLNODE04, synchronous commit, automatic failover |
| **DAG1** | Distributed AG linking AG1 to AG2, asynchronous commit, manual failover, endpoints through the AG listeners |
| **MGMT01** | Machine the scripts run from. Not a SQL node (in the lab this was the domain controller) |

Replace them with your own.

> [!NOTE]
> **Before you start**
>
> - An approved change, if this is not a lab
> - Local administrator on every node and **sysadmin** on every instance, from the account you run the scripts as
> - PowerShell remoting (WinRM) enabled on every node
> - The scripts saved together in one folder on MGMT01, for example `C:\Scripts`: [Invoke-SQLPatching.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Invoke-SQLPatching.ps1), [Invoke-AGPatching.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Invoke-AGPatching.ps1) and [Get-AGStatusReport.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Get-AGStatusReport.ps1)
> - The port every instance listens on (1433 unless you changed it)
> - Each AG has two nodes. The scripts handle one primary and one secondary per AG

> [!IMPORTANT]
> **Run the scripts from MGMT01, never from a node.** They restart SQL Server and reboot the nodes. Run on a node, a script would reboot the machine it is running on halfway through, with an AG failed over and nothing left to fail it back. The scripts check for this and refuse to start.

## Step 1: Check the current builds and AG health

Open an elevated PowerShell window on MGMT01 and define the AGs once. Every later step reuses `$ags`.

```powershell
# On MGMT01
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
cd C:\Scripts
$ags = @{ Name = 'AG1'; Nodes = 'SQLNODE01','SQLNODE02' }, @{ Name = 'AG2'; Nodes = 'SQLNODE03','SQLNODE04' }
.\Get-AGStatusReport.ps1 -AvailabilityGroups $ags -DagName 'DAG1' -SqlPort 1433
```

Then compare each node's build with the latest on the [SQL Server 2022 build versions](https://learn.microsoft.com/en-us/troubleshoot/sql/releases/sqlserver-2022/build-versions) page:

```sql
-- On each node, or from a multi-server query window
SELECT @@SERVERNAME AS ServerName, SERVERPROPERTY('ProductVersion') AS Build, @@VERSION AS VersionText;
```

> [!TIP]
> **Expected result:** The report ends with **Overall: HEALTHY**. Local AG databases are SYNCHRONIZED, and the DAG databases are SYNCHRONIZING, which is the healthy state for an asynchronous link. In the lab three nodes were on **16.0.4265.3** (CU26) and one was already on **16.0.4305.5** (CU27 plus the October 2026 security update, KB5126934).

## Step 2: Stop Windows Update installing SQL Server updates by itself

If **Give me updates for other Microsoft products** is on, anyone clicking Install in Windows Update on a node also installs SQL Server updates on that one node, outside the rolling order and possibly on a primary. That is how one lab node ended up ahead of the others. Check every node from MGMT01:

```powershell
# On MGMT01
Invoke-Command -ComputerName SQLNODE01, SQLNODE02, SQLNODE03, SQLNODE04 -ScriptBlock {
    $p  = Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU' -ErrorAction SilentlyContinue
    $mu = (New-Object -ComObject Microsoft.Update.ServiceManager).Services | Where-Object Name -eq 'Microsoft Update'
    [pscustomobject]@{ AUOptions = $p.AUOptions; MicrosoftUpdate = [bool]$mu }
} | Select-Object PSComputerName, AUOptions, MicrosoftUpdate | Format-Table -AutoSize
```

On any node showing `MicrosoftUpdate = True`, go to **Settings > Update & Security > Windows Update > Advanced options** and untick **Give me updates for other Microsoft products when I update Windows**.

> [!TIP]
> **Expected result:** `MicrosoftUpdate = False` on every node. `AUOptions = 3` (download automatically, notify to install) is a good setting for SQL nodes: updates are pre-downloaded, and nothing installs or reboots until a script or a person says so. From here on, don't click Install in Windows Update on the nodes. Patch only through the scripts.

## Step 3: Download the SQL Server update

On MGMT01, open the [Microsoft Update Catalog](https://www.catalog.update.microsoft.com/), search for the KB (for example **KB5126934**), click **Download** and save the `.exe` into a folder such as `C:\Patches\SQL`. A security update for a CU already includes that CU, so one file is enough. Give it a short name:

```powershell
# On MGMT01
New-Item -Path C:\Patches\SQL -ItemType Directory -Force | Out-Null
Move-Item "$env:USERPROFILE\Downloads\sqlserver2022-kb5126934-x64_*.exe" 'C:\Patches\SQL\SQLServer2022-KB5126934-x64.exe'
Get-ChildItem C:\Patches\SQL
```

> [!TIP]
> **Expected result:** One file of roughly 480 MB. Note the build it installs from the KB article (16.0.4305.5 for KB5126934). That is your `-TargetBuild`.

## Step 4: Dry run the SQL patching

Without `-Execute` the script changes nothing. It reads every build, works out the order and says what it would do.

```powershell
# On MGMT01
.\Invoke-SQLPatching.ps1 -AvailabilityGroups $ags -DagName 'DAG1' -SqlPort 1433 `
    -InstallerPath 'C:\Patches\SQL\SQLServer2022-KB5126934-x64.exe' -TargetBuild 16.0.4305.5
```

> [!TIP]
> **Expected result:** `Patch order: AG2 (DAG SECONDARY) -> AG1 (DAG PRIMARY)`, every AG healthy, and a summary showing **Will patch** or **Skip (already patched)** for each node. If a connection fails with *Named Pipes Provider, error: 40*, the `-SqlPort` value is wrong. See Lessons from the lab.

## Step 5: Run the SQL patching

Close SSMS on the nodes, then run the same command with `-Execute`. Leave the window open until the summary appears.

```powershell
# On MGMT01
.\Invoke-SQLPatching.ps1 -AvailabilityGroups $ags -DagName 'DAG1' -SqlPort 1433 `
    -InstallerPath 'C:\Patches\SQL\SQLServer2022-KB5126934-x64.exe' -TargetBuild 16.0.4305.5 -Execute
```

What happens, in order:

1. **AG2 (DAG secondary side):** patch SQLNODE04, confirm the build and resync, fail AG2 over to SQLNODE04, patch SQLNODE03, fail back.
2. **Safety check:** every AG2 node must be on the target build before AG1 is touched. A newer build must never become primary while older replicas still receive from it.
3. **AG1 (global primary side):** the same, secondary first. A node already on the target build is skipped.

> [!TIP]
> **Expected result:** Each node shows `Setup exit code: 0` and `Now on 16.0.4305.5`, and every failover is followed by `healthy ... OK`. The lab run took about **24 minutes** for three nodes, about 4 to 5 minutes of setup each, with **no reboots**: setup only restarted the SQL Server service.

## Step 6: Validate and clean up

The script ends with a before/after table and the AG status report. Then remove the copied installer from each node:

```powershell
# On MGMT01
Invoke-Command -ComputerName SQLNODE01, SQLNODE02, SQLNODE03, SQLNODE04 -ScriptBlock {
    Remove-Item C:\Temp\SQLPatch -Recurse -Force -ErrorAction SilentlyContinue
}
```

> [!TIP]
> **Expected result:** Every node on the target build, SQLNODE01 and SQLNODE03 back as primaries, and **Overall: HEALTHY**. Logs are in `C:\Scripts\Logs_SQL` and the HTML report in `C:\Scripts\Logs_AGStatus`.

## Step 7: Dry run the Windows patching

Run this any day after Patch Tuesday (the second Tuesday of the month), and not in the same window as the SQL patching, so each change stands on its own.

```powershell
# On MGMT01
.\Invoke-AGPatching.ps1 -AvailabilityGroups $ags -DagName 'DAG1' -SqlPort 1433
```

The scan runs on each node through a one-time scheduled task as SYSTEM, because the Windows Update API refuses to download or install from a remote PowerShell session. Each node takes about a minute.

> [!TIP]
> **Expected result:** The same patch order, every AG healthy, and the number of updates waiting on each node. SQL Server updates and Defender definition updates are excluded by default. Defender installs its own definitions several times a day without a reboot, and including them would fail both AGs over on every run.

## Step 8: Run the Windows patching

```powershell
# On MGMT01
.\Invoke-AGPatching.ps1 -AvailabilityGroups $ags -DagName 'DAG1' -SqlPort 1433 -Execute
```

Same order as Step 5. A node is rebooted only if Windows says it needs one. If the primary has nothing to install, there is no failover for that AG.

> [!TIP]
> **Expected result:** Every node `OK`, the original primaries back in place, and **Overall: HEALTHY** at the end. Logs are in `C:\Scripts\Logs_Patch`.

## Step 9 (optional): Keep SSMS current on the nodes

[Update-SSMS.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Update-SSMS.ps1) downloads the latest SSMS 22 bootstrapper and updates every server silently. [Register-SSMSAutoUpdate.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Register-SSMSAutoUpdate.ps1) puts that on a schedule. SSMS is a client tool, so this doesn't need the rolling order.

```powershell
# On MGMT01: report, update now, then schedule every 3 weeks
.\Update-SSMS.ps1 -ComputerName SQLNODE01, SQLNODE02, SQLNODE03, SQLNODE04
.\Update-SSMS.ps1 -ComputerName SQLNODE01, SQLNODE02, SQLNODE03, SQLNODE04 -Execute -Force
.\Register-SSMSAutoUpdate.ps1 -ComputerName SQLNODE01, SQLNODE02, SQLNODE03, SQLNODE04
```

> [!TIP]
> **Expected result:** In the lab all four went from 22.9.0 to 22.11.3. One returned exit code 3010 (success, reboot required) and kept reporting it until it was restarted.

## Validation sign-off

- [ ] Every node on the target SQL Server build
- [ ] Windows updates installed on every node, no reboot pending
- [ ] Original primaries back in place (or the new layout recorded, if `-NoFailBack` was used)
- [ ] AG status report shows **Overall: HEALTHY**
- [ ] Logs from `Logs_SQL`, `Logs_Patch` and `Logs_AGStatus` attached to the change record
- [ ] `MicrosoftUpdate = False` on every node
- [ ] Installer copies removed from `C:\Temp\SQLPatch`

## Lessons from the lab

- **Non-default port.** The instances listened on a custom port only. Connecting by server name tried 1433, failed, and fell back to Named Pipes, which worked one moment and failed with *error: 40* the next. The scripts now connect to `tcp:<node>,<port>`, so pass `-SqlPort`. Check the port from MGMT01 with `Test-NetConnection SQLNODE01 -Port <port>`.
- **One node already ahead.** Windows Update had installed a SQL security update on one secondary on its own (Step 2). It was harmless because that node was a secondary. The script skipped it and made sure the DAG secondary side was fully patched before failing the global primary side over to the newer build.
- **Defender definitions.** The first Windows dry run found an update on every node: a Defender Security Intelligence Update. Left in, it would have failed both AGs over on every run for an update that needs no reboot. Excluded by default.
- **Management machine memory.** The DC running the scripts had 2 GB and sat at 95% memory, swapping. Give the machine that runs the scripts enough memory (4 GB was comfortable) so it doesn't struggle halfway through a failover.
- **Run long jobs in a plain console.** Edit in ISE or VS Code if you like, but run the patching in a regular elevated PowerShell window, the same way a scheduled task would run it.

## Sources

- **Microsoft Learn:** [Upgrade availability group replicas](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/upgrading-always-on-availability-group-replica-instances)
- **Microsoft Learn:** [SQL Server 2022 build versions](https://learn.microsoft.com/en-us/troubleshoot/sql/releases/sqlserver-2022/build-versions)
- **Microsoft Learn:** [Latest updates and version history for SQL Server](https://learn.microsoft.com/en-us/troubleshoot/sql/releases/download-and-install-latest-updates)
- **Microsoft Support:** [KB5126934: Security update for SQL Server 2022 CU27, October 13, 2026](https://support.microsoft.com/en-us/servicing/sql/sql-server-2022/cumulative-update/kb5126934-cu27-gdr-october-2026)
- **Microsoft Learn:** [SSMS command-line parameters](https://learn.microsoft.com/ssms/install/command-line-parameters)

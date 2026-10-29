# Move PowerShell scripts and Agent jobs from SQLPS to the SqlServer module

**Status:** Draft · **Applies to:** SQL Server 2016 / 2019 / 2022, Windows PowerShell 5.1 · **Steps:** 7

Replace the old **SQLPS** module with the **SqlServer** module in PowerShell scripts and SQL Server Agent jobs. SQLPS ships with SQL Server but is no longer updated. SqlServer is the supported module, gets new cmdlets and fixes, and is installed from the PowerShell Gallery.

In this runbook **DEVSQL01** is the server being changed. Replace it with your own, and start on Dev.

> [!NOTE]
> **Before you start**
>
> - An approved change (on Dev, a service request is usually enough; in Prod, a change ticket covering every job you modify)
> - Local administrator on the server, and an **elevated** PowerShell window
> - `sysadmin` or `SQLAgentOperatorRole` plus job ownership to edit Agent job steps
> - A list of the folders where your PowerShell scripts live

## Step 1: Find what still uses SQLPS

Search your script folders for anything that imports SQLPS or uses the old `cd SQLSERVER:` workaround.

```powershell
# On DEVSQL01. Change the paths to your script folders.
Get-ChildItem -Path 'D:\Scripts', 'C:\Scripts' -Recurse -Include *.ps1, *.psm1 -ErrorAction SilentlyContinue |
    Select-String -Pattern 'SQLPS', 'Push-Location', 'Pop-Location', 'DisableNameChecking' |
    Select-Object Path, LineNumber, Line
```

Then list the Agent job steps that run PowerShell or mention SQLPS.

```sql
-- On DEVSQL01
SELECT  j.name        AS job_name,
        s.step_id,
        s.step_name,
        s.subsystem,
        s.command
FROM    msdb.dbo.sysjobs      AS j
JOIN    msdb.dbo.sysjobsteps  AS s ON s.job_id = j.job_id
WHERE   s.subsystem = 'PowerShell'
   OR   s.command LIKE '%SQLPS%'
ORDER BY j.name, s.step_id;
```

<!-- ![Step 1: Find what still uses SQLPS](../images/migrate-sqlps-to-sqlserver-module/step-01.png) -->

> [!TIP]
> **Expected result:** A list of script files and job steps to change. Save it with the change record so nothing gets missed.

## Step 2: Install the SqlServer module

If the server can reach the PowerShell Gallery:

```powershell
# On DEVSQL01, elevated
Install-Module SqlServer -Scope AllUsers -AllowClobber
```

`-AllowClobber` is needed because some cmdlet names also exist in SQLPS.

If the server has no internet access, download it on a machine that does and copy it across:

```powershell
# On a machine with internet access
Save-Module SqlServer -Path C:\Temp\Modules

# Copy C:\Temp\Modules\SqlServer to this folder on DEVSQL01:
# C:\Program Files\WindowsPowerShell\Modules\SqlServer
```

<!-- ![Step 2: Install the SqlServer module](../images/migrate-sqlps-to-sqlserver-module/step-02.png) -->

> [!TIP]
> **Expected result:** `Get-Module SqlServer -ListAvailable` shows the module and its version.

## Step 3: Update the import line in each script

Replace the old import:

```powershell
# Old
Push-Location
Import-Module SQLPS -DisableNameChecking
Pop-Location
```

with:

```powershell
# New
Remove-Module SQLPS -ErrorAction SilentlyContinue   # avoids cmdlet conflicts if SQLPS was already loaded
Import-Module SqlServer
```

The `Push-Location` / `Pop-Location` lines can go. SQLPS changed the current location to the `SQLSERVER:\` drive when it loaded, which broke relative paths. SqlServer doesn't do that. The `SQLSERVER:\` provider still exists if a script uses it on purpose.

<!-- ![Step 3: Update the import line in each script](../images/migrate-sqlps-to-sqlserver-module/step-03.png) -->

> [!TIP]
> **Expected result:** No script in the Step 1 list still references `SQLPS`.

## Step 4: Fix Invoke-Sqlcmd connections (version 22 and later)

From SqlServer **22.0**, `Invoke-Sqlcmd` encrypts connections by default (`-Encrypt Mandatory`). Instances with a self-signed certificate, or no certificate, will fail with a certificate trust error.

Pick one per script:

```powershell
# Trust the server certificate (still encrypted)
Invoke-Sqlcmd -ServerInstance DEVSQL01 -Query 'SELECT @@VERSION;' -TrustServerCertificate

# Or turn mandatory encryption off for older instances
Invoke-Sqlcmd -ServerInstance DEVSQL01 -Query 'SELECT @@VERSION;' -Encrypt Optional
```

If you need time to fix every script, pin the last 21.x release, which keeps the old behaviour:

```powershell
Import-Module SqlServer -RequiredVersion 21.1.18256
```

<!-- ![Step 4: Fix Invoke-Sqlcmd connections](../images/migrate-sqlps-to-sqlserver-module/step-04.png) -->

> [!TIP]
> **Expected result:** Each script connects and returns results with no certificate errors.

## Step 5: Change PowerShell job steps to CmdExec

The Agent **PowerShell** step type runs inside the SQLPS host, so it can't use SqlServer cleanly. Move the code into a `.ps1` file and change the step type to **Operating system (CmdExec)**:

```bat
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "D:\Scripts\JobName.ps1"
```

Put `Import-Module SqlServer` at the top of the `.ps1` file. The step now runs as the SQL Agent service account (or a proxy), so check that account can read the script folder.

Add `exit 1` (or `throw`) in the script's error handling. Without it, a failed script can still report success to Agent.

<!-- ![Step 5: Change PowerShell job steps to CmdExec](../images/migrate-sqlps-to-sqlserver-module/step-05.png) -->

> [!TIP]
> **Expected result:** The job step shows type **Operating system (CmdExec)** and the command calls the `.ps1` file.

## Step 6: Run each job and script on Dev

Start each changed job by hand and read the step history.

```sql
-- On DEVSQL01
EXEC msdb.dbo.sp_start_job @job_name = N'JobName';

-- After it finishes
SELECT TOP (5) j.name, h.step_id, h.run_status, h.message, h.run_date, h.run_time
FROM   msdb.dbo.sysjobhistory AS h
JOIN   msdb.dbo.sysjobs       AS j ON j.job_id = h.job_id
WHERE  j.name = N'JobName'
ORDER BY h.instance_id DESC;
```

`run_status = 1` means success. Also force one failure (for example, point the script at a server that doesn't exist) and confirm the job reports it as failed.

<!-- ![Step 6: Run each job and script on Dev](../images/migrate-sqlps-to-sqlserver-module/step-06.png) -->

> [!TIP]
> **Expected result:** Every changed job succeeds, and the forced failure shows as a failed step.

## Step 7: Roll out to the other servers

Repeat Steps 2 to 6 on each server, Dev first, then QA, then Prod. On availability groups, change the job on **every replica**, because Agent jobs aren't synchronized between replicas.

> [!TIP]
> **Expected result:** Step 1 returns nothing on every server.

## Validation sign-off

- [ ] `Get-Module SqlServer -ListAvailable` shows the module on every server
- [ ] No script or job step from Step 1 still references `SQLPS`
- [ ] No job step uses the **PowerShell** subsystem
- [ ] Every changed job ran successfully, and a forced failure was reported as failed
- [ ] On availability groups, jobs match on every replica

## Sources

- **Microsoft Learn:** [Install the SQL Server PowerShell module](https://learn.microsoft.com/en-us/powershell/sql-server/download-sql-server-ps-module)
- **Microsoft Learn:** [SQL Server PowerShell](https://learn.microsoft.com/en-us/powershell/sql-server/sql-server-powershell)
- **Microsoft Learn:** [Run Windows PowerShell steps in SQL Server Agent](https://learn.microsoft.com/en-us/sql/powershell/run-windows-powershell-steps-in-sql-server-agent)

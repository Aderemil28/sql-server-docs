# Find when an instance was upgraded, without the change ticket

**Status:** Draft · **Applies to:** SQL Server 2016 to 2022 · **Steps:** 6

Work out when an instance moved from one SQL Server version to another, and whether it was an in-place upgrade, a side-by-side migration or a rebuild, using only what is on the server. Use it when the change ticket can't be found or the history is unclear.

> [!NOTE]
> **Before you start**
>
> - sysadmin on the instance, and local admin on the server (RDP) for the PowerShell steps
> - Run PowerShell locally on the server. Remote shell may be disabled.
> - Don't trust a single date. The answer comes from lining up all six steps.

## Step 1: Instance install date

The `NT AUTHORITY\SYSTEM` login is created when the instance is installed.

```sql
-- Run on the instance
SELECT create_date AS instance_install_date
FROM sys.server_principals
WHERE sid = 0x010100000000000512000000;
```

> [!TIP]
> **Expected result:** One date. An in-place upgrade keeps the original install date, so on its own this does not tell you when the upgrade happened. A reinstall resets it.

## Step 2: Database dates and compatibility levels

```sql
SELECT name, create_date, compatibility_level
FROM sys.databases
ORDER BY create_date;

SELECT destination_database_name, restore_date, user_name
FROM msdb.dbo.restorehistory
ORDER BY restore_date;
```

> [!TIP]
> **Expected result:** master and model always show 2003; ignore them. msdb's date is stamped by the version that installed it, so a 2019-era msdb date means the instance was installed fresh on 2019. User databases dated after the install date were loaded later. An empty `restorehistory` usually means the history cleanup job purged it, not that nothing was restored.

## Step 3: Which version each database was born on

```sql
-- Change the database name, then run on the instance
CREATE TABLE #dbinfo (ParentObject nvarchar(255), Object nvarchar(255), Field nvarchar(255), Value nvarchar(255));
INSERT #dbinfo EXEC('DBCC DBINFO(''YourDatabase'') WITH TABLERESULTS');
SELECT Field, Value FROM #dbinfo
WHERE Field IN ('dbi_createVersion', 'dbi_version', 'dbi_crdate');
DROP TABLE #dbinfo;
```

| Internal version | SQL Server |
|---|---|
| 655 | 2008 |
| 660 / 661 | 2008 R2 |
| 706 | 2012 |
| 782 | 2014 |
| 852 | 2016 |
| 869 | 2017 |
| 904 | 2019 |
| 957 | 2022 |

> [!TIP]
> **Expected result:** `dbi_version` matches the current instance. `dbi_createVersion` shows where the database was first created, which can be several versions back. It does not show every hop in between.

## Step 4: OS build date and SQL install dates from the registry

```powershell
# Run locally on the server
(Get-CimInstance Win32_OperatingSystem).InstallDate

Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\* |
  Where-Object DisplayName -like "SQL Server 20*Database Engine*" |
  Select-Object DisplayName, InstallDate
```

> [!TIP]
> **Expected result:** The OS date tells you when the server was built. Registry `InstallDate` (yyyyMMdd) entries also update when a CU is applied, so a recent date there is usually patching, not a new install.

## Step 5: Every Setup run, with its action and result

This is the most reliable evidence. Every install, upgrade, patch and uninstall leaves a dated folder.

```powershell
# Run locally on the server
Get-ChildItem "C:\Program Files\Microsoft SQL Server\*\Setup Bootstrap\Log\*\Summary*.txt" |
  ForEach-Object {
    $_.FullName
    Select-String -Path $_.FullName -Pattern 'Action:|Start time|Final result'
  }
```

> [!TIP]
> **Expected result:** Look for `Requested action: Upgrade`, `Install`, `Uninstall` and `Patch`, each with a start time and final result. Logs from an older version's folder (for example `140`) may be gone if that version was uninstalled, so a missing folder is not proof the version was never there.

## Step 6: Confirm the version it upgraded from

Open the Summary file from the `Upgrade` run found in Step 5.

```powershell
# Change the path to the Upgrade run's folder
Get-Content "C:\Program Files\Microsoft SQL Server\150\Setup Bootstrap\Log\<yyyyMMdd_HHmmss>\Summary_<MACHINE>_<yyyyMMdd_HHmmss>.txt" -TotalCount 40
```

> [!TIP]
> **Expected result:** Under **Product features discovered**, the Database Engine Services row shows the pre-upgrade version and edition (14.0.x = 2017, 13.0.x = 2016). **Package properties** shows the version it was upgraded to.

## Reading the results together

| What you see | What it means |
|---|---|
| `Upgrade` run in Setup logs | In-place upgrade on that date |
| `Uninstall` then `Install` a few days later | Instance was rebuilt clean after the upgrade |
| OS install date close to the SQL install date | New server, side-by-side migration |
| User databases dated after the install date | Data was restored or attached after the build |
| Recent registry date with a `Patch` run on the same day | A CU, not a reinstall |

Write the finding up in one or two lines: what changed, when, and that the source was the Setup Bootstrap logs.

## Validation sign-off

- [ ] Upgrade date taken from the Setup Bootstrap logs, not from a single DMV
- [ ] Pre-upgrade version confirmed from the Upgrade run's Summary file
- [ ] Any uninstall or reinstall after the upgrade accounted for
- [ ] Finding recorded where the team will look for it

## Sources

- **Microsoft Learn:** [View and read SQL Server Setup log files](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/view-and-read-sql-server-setup-log-files)
- **Microsoft Learn:** [restorehistory (Transact-SQL)](https://learn.microsoft.com/en-us/sql/relational-databases/system-tables/restorehistory-transact-sql)
- **Microsoft Learn:** [sys.databases (Transact-SQL)](https://learn.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-databases-transact-sql)

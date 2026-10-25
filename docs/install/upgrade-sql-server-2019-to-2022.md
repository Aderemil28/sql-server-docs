# In-place upgrade: SQL Server 2019 to SQL Server 2022 (standalone instance)

**Status:** Draft · **Applies to:** Standalone SQL Server 2019 instances on Windows Server 2016 or later · **Steps:** 17

Upgrade a standalone SQL Server 2019 Database Engine instance to SQL Server 2022 in place, on the same server, keeping the instance name, databases, logins, jobs and settings. The runbook covers the checks done weeks ahead, the change window itself, validation after the upgrade, and how to roll back.

In this runbook **SQL01** is the server being upgraded. **AppDB** stands for any user database, and **X:\SQLBackups\PreUpgrade** is where the pre-upgrade backups go. Replace these with your own names.

> [!NOTE]
> **Before you start**
>
> - The instance is **standalone**: not a failover cluster instance and not an Always On availability group replica. Those need a rolling upgrade, which is a different runbook
> - Windows Server 2016 or later. SQL Server 2022 does not run on Windows Server 2012 R2, so upgrade the OS first if needed
> - SQL Server 2022 licence and media for the **same edition** as today (Enterprise stays Enterprise, Standard can stay Standard or go to Enterprise)
> - The application vendor supports SQL Server 2022 in writing
> - Local administrator on SQL01 and sysadmin on the instance
> - Space for a full backup of every database, plus the 6 GB that Setup needs free on the C: drive
> - An approved change with a downtime window (allow 1 to 2 hours for the upgrade itself, plus testing time)
> - A **point of no return** agreed with the application owner (see Step 17): after the application starts writing to the upgraded instance, rolling back loses that data

> [!IMPORTANT]
> **There is no downgrade.** After the upgrade, every database is upgraded to the SQL Server 2022 internal format and can't be restored or attached to SQL Server 2019. Your rollback is the VM snapshot or the backups taken in Step 9.

## Timeline

| When | Steps |
|---|---|
| Two or more weeks before | 1 to 6: readiness checks, blockers, vendor sign-off, Query Store baseline, media |
| The day before | 7 to 8: script out configuration, back up keys, confirm CHECKDB |
| Change window | 9 to 14: stop the application, back up, snapshot, upgrade, validate |
| After the window | 15: hand back and monitor |
| A later change | 16: raise the compatibility level to 160 |
| Only if needed | 17: roll back |

## Step 1: Confirm the instance can be upgraded

Run on SQL01. This confirms the version, edition, that the instance is standalone, and the state of every database.

```sql
-- Run on the SQL Server 2019 instance
SELECT  SERVERPROPERTY('ServerName')          AS server_name,
        SERVERPROPERTY('ProductVersion')      AS product_version,   -- 15.0.x = SQL Server 2019
        SERVERPROPERTY('ProductUpdateLevel')  AS cumulative_update,
        SERVERPROPERTY('Edition')             AS edition,
        SERVERPROPERTY('IsClustered')         AS is_clustered,      -- must be 0
        SERVERPROPERTY('IsHadrEnabled')       AS is_hadr_enabled,   -- must be 0
        SERVERPROPERTY('IsFullTextInstalled') AS fulltext_installed,
        SERVERPROPERTY('IsPolyBaseInstalled') AS polybase_installed;

SELECT  name, state_desc, compatibility_level, recovery_model_desc,
        is_read_only, SUSER_SNAME(owner_sid) AS owner_name
FROM sys.databases
ORDER BY name;
```

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-01.png and uncomment the next line -->
<!-- ![Step 1: Confirm the instance can be upgraded](../images/upgrade-sql-server-2019-to-2022/step-01.png) -->

> [!TIP]
> **Expected result:** ProductVersion starts with 15.0, IsClustered and IsHadrEnabled are 0, the edition matches your SQL Server 2022 licence, and every database is ONLINE. Fix or explain any database that is OFFLINE, SUSPECT or RECOVERY_PENDING before going further.

## Step 2: Check the server (OS, .NET, disk, pending restart)

Run in an elevated PowerShell window on SQL01.

```powershell
# OS: Windows Server 2016 (10.0.14393) or later
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, LastBootUpTime

# .NET Framework 4.7.2 or later = Release 461808 or higher
(Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full').Release

# Setup needs at least 6 GB free on the system drive
Get-PSDrive C | Select-Object @{n='FreeGB';e={[math]::Round($_.Free/1GB,1)}}

# A pending restart blocks Setup. Any True here means restart first.
Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending'
Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired'

# Windows Installer must not be disabled
Get-Service msiserver | Select-Object Name, Status, StartType

# SQL services on this server today
Get-Service | Where-Object { $_.Name -like 'MSSQL*' -or $_.Name -like 'SQL*' } |
    Select-Object Name, DisplayName, Status, StartType
```

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-02.png and uncomment the next line -->
<!-- ![Step 2: Check the server](../images/upgrade-sql-server-2019-to-2022/step-02.png) -->

> [!TIP]
> **Expected result:** OS is Windows Server 2016 or later, the .NET Release value is 461808 or higher, C: has more than 6 GB free, both restart checks return False, and msiserver is not Disabled.

## Step 3: Find features that change or block the upgrade

Some features are removed or behave differently in SQL Server 2022. Run on SQL01 and deal with every row that comes back before the window.

```sql
-- 3a. Linked servers using SQL Server Native Client (SQLNCLI). Native Client does not ship
--     with SQL Server 2022. Move them to MSOLEDBSQL (see the linked server runbook).
SELECT name, provider, data_source
FROM sys.servers
WHERE is_linked = 1 AND provider LIKE N'SQLNCLI%';

-- 3b. Machine Learning Services. SQL Server 2022 Setup no longer installs the R and Python
--     runtimes; they must be installed separately after the upgrade.
SELECT name, value_in_use FROM sys.configurations WHERE name = N'external scripts enabled';
SELECT servicename, status_desc FROM sys.dm_server_services WHERE servicename LIKE N'%Launchpad%';

-- 3c. Stretch Database (deprecated in SQL Server 2022)
SELECT name FROM sys.databases WHERE is_remote_data_archive_enabled = 1;

-- 3d. PolyBase Hadoop external data sources (Hadoop connectivity is removed in SQL Server 2022)
DECLARE @sql nvarchar(max);
SELECT @sql = STRING_AGG(CAST(N'SELECT N' + QUOTENAME(name, '''') + N' AS database_name, name, location
    FROM ' + QUOTENAME(name) + N'.sys.external_data_sources WHERE type_desc = N''HADOOP''' AS nvarchar(max)),
    N' UNION ALL ')
FROM sys.databases
WHERE database_id > 4 AND state_desc = N'ONLINE';
IF @sql IS NOT NULL EXEC sys.sp_executesql @sql;

-- 3e. Replication and log shipping. For log shipping, upgrade the secondary before the primary.
SELECT name, is_published, is_subscribed, is_merge_published, is_distributor
FROM sys.databases
WHERE is_published = 1 OR is_subscribed = 1 OR is_merge_published = 1 OR is_distributor = 1;
SELECT primary_database FROM msdb.dbo.log_shipping_primary_databases;
SELECT secondary_database FROM msdb.dbo.log_shipping_secondary_databases;

-- 3f. SSIS catalog (needs its own upgrade afterwards, Step 14)
SELECT name FROM sys.databases WHERE name = N'SSISDB';

-- 3g. Deprecated features used since the last restart. Pass the list to the application team.
SELECT RTRIM(instance_name) AS deprecated_feature, cntr_value AS times_used
FROM sys.dm_os_performance_counters
WHERE object_name LIKE N'%Deprecated Features%' AND cntr_value > 0
ORDER BY cntr_value DESC;

-- 3h. SQL Server Agent service account must be sysadmin for the upgrade
SELECT servicename, service_account,
       IS_SRVROLEMEMBER(N'sysadmin', service_account) AS is_sysadmin   -- NULL = no login by that name
FROM sys.dm_server_services
WHERE servicename LIKE N'SQL Server Agent%';
```

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-03.png and uncomment the next line -->
<!-- ![Step 3: Find features that change or block the upgrade](../images/upgrade-sql-server-2019-to-2022/step-03.png) -->

> [!TIP]
> **Expected result:** 3a to 3e return no rows, or each row has an owner and a plan. 3f tells you whether Step 14 needs the SSISDB upgrade. 3g is kept for the application team. 3h shows is_sysadmin = 1 (if it shows NULL, the Agent account gets sysadmin through a group: confirm that group is in the sysadmin role).

## Step 4: Confirm everything that connects supports SQL Server 2022

The upgrade only succeeds if everything around the instance still works afterwards. Get a yes for each item and record it in the change:

| Item | What to confirm |
|---|---|
| Application | Vendor or internal team supports SQL Server 2022, in writing |
| Monitoring | Your monitoring tools (for example Idera SQL Diagnostic Manager and the SCOM SQL Server management pack) support SQL Server 2022 at the versions you run |
| Backup software | Any third-party backup agent supports SQL Server 2022 |
| Scheduler and ETL | Job schedulers, SSIS packages and ETL tools that connect to this instance |
| Client tools | DBAs use SSMS 20 or later to manage SQL Server 2022 |
| Security tooling | Application control, antivirus exclusions and vulnerability scanners that reference SQL Server 2019 paths (the 2022 binaries live under `MSSQL16.<instance>`) |

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-04.png and uncomment the next line -->
<!-- ![Step 4: Confirm everything that connects supports SQL Server 2022](../images/upgrade-sql-server-2019-to-2022/step-04.png) -->

> [!TIP]
> **Expected result:** Every row has a confirmed yes and a name against it in the change record.

## Step 5: Turn on Query Store and capture a baseline

The upgrade keeps each database at its current compatibility level (150), so query plans should not change much on the day. You will raise the level to 160 later (Step 16), and for that you need to know how queries performed **before**. Turn on Query Store now, at least one full business cycle (for example a month end) before the upgrade.

```sql
-- Run on SQL01. Lists the statements first; set @Execute = 1 to run them.
DECLARE @Execute bit = 0;
DECLARE @sql nvarchar(max);

SELECT @sql = STRING_AGG(CAST(N'ALTER DATABASE ' + QUOTENAME(name)
       + N' SET QUERY_STORE = ON (OPERATION_MODE = READ_WRITE);' AS nvarchar(max)), CHAR(10))
FROM sys.databases
WHERE database_id > 4 AND state_desc = N'ONLINE' AND is_read_only = 0 AND is_query_store_on = 0;

PRINT ISNULL(@sql, N'-- Query Store is already on for every user database');
IF @Execute = 1 AND @sql IS NOT NULL EXEC sys.sp_executesql @sql;

-- Wait statistics since the last restart. Save this output as the pre-upgrade baseline.
SELECT TOP (20) wait_type, waiting_tasks_count, wait_time_ms, signal_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type NOT LIKE N'SLEEP%' AND wait_type NOT LIKE N'%IDLE%' AND wait_type NOT LIKE N'%QUEUE%'
  AND wait_type NOT IN (N'BROKER_TASK_STOP', N'CHECKPOINT_QUEUE', N'DIRTY_PAGE_POLL',
                        N'HADR_FILESTREAM_IOMGR_IOCOMPLETION', N'SP_SERVER_DIAGNOSTICS_SLEEP',
                        N'WAITFOR', N'XE_TIMER_EVENT', N'XE_DISPATCHER_WAIT')
ORDER BY wait_time_ms DESC;
```

Also save the output of [top-cpu-queries.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/monitoring/top-cpu-queries.sql) from the toolkit.

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-05.png and uncomment the next line -->
<!-- ![Step 5: Turn on Query Store and capture a baseline](../images/upgrade-sql-server-2019-to-2022/step-05.png) -->

> [!TIP]
> **Expected result:** Query Store is on (READ_WRITE) for every user database, and the wait stats and top CPU outputs are saved with the change.

## Step 6: Get the media and the latest cumulative update ready

1. Copy the SQL Server 2022 ISO for your edition (from your volume licensing portal) to SQL01, for example `C:\SQLInstall\SQL2022`
2. Download the **latest SQL Server 2022 cumulative update** from the Microsoft "SQL Server 2022 build versions" page and put only that file in its own folder, for example `C:\SQLUpdates\2022`. Setup will apply it during the upgrade, so the instance comes up patched
3. Check the CU against your patching standard: it should be the same CU your other SQL Server 2022 servers run, unless the change says otherwise

```powershell
# Confirm the files are where Setup will look for them
Get-ChildItem C:\SQLInstall\SQL2022, C:\SQLUpdates\2022 | Select-Object FullName, Length, LastWriteTime
```

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-06.png and uncomment the next line -->
<!-- ![Step 6: Get the media and the latest cumulative update ready](../images/upgrade-sql-server-2019-to-2022/step-06.png) -->

> [!TIP]
> **Expected result:** The ISO and exactly one CU file are on SQL01, and the CU matches your patching standard.

## Step 7: Script out the configuration and back up keys (the day before)

If anything goes wrong you want to rebuild this instance exactly. Save everything below to a folder **off the server** and attach it to the change.

```sql
-- Instance settings
SELECT name, value, value_in_use FROM sys.configurations ORDER BY name;

-- Trace flags and startup parameters
DBCC TRACESTATUS(-1);
SELECT value_name, value_data FROM sys.dm_server_registry WHERE value_name LIKE N'SQLArg%';

-- Certificates in master (TDE and backup encryption) and when their private keys were last backed up
SELECT name, subject, expiry_date, pvt_key_last_backup_date
FROM master.sys.certificates
WHERE name NOT LIKE N'##%';

-- Databases encrypted with TDE (encryption_state 3 = encrypted)
SELECT DB_NAME(database_id) AS database_name, encryption_state, key_algorithm, key_length
FROM sys.dm_database_encryption_keys;
```

Back up the service master key and every certificate that protects a database. Store the files and passwords in your password vault, not on SQL01:

```sql
USE master;
BACKUP SERVICE MASTER KEY TO FILE = N'X:\SQLBackups\PreUpgrade\SQL01_SMK.key'
    ENCRYPTION BY PASSWORD = N'<StrongPassword>';

-- Repeat for each certificate listed above
BACKUP CERTIFICATE TDECert TO FILE = N'X:\SQLBackups\PreUpgrade\TDECert.cer'
    WITH PRIVATE KEY (FILE = N'X:\SQLBackups\PreUpgrade\TDECert.pvk',
                      ENCRYPTION BY PASSWORD = N'<StrongPassword>');
```

Then script out, with the toolkit or SSMS:

- Logins with passwords and SIDs: [script-out-logins-sp_help_revlogin.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/security/script-out-logins-sp_help_revlogin.sql)
- Agent jobs, operators and alerts: in SSMS open **SQL Server Agent > Jobs**, press F7, select all jobs, right-click **Script Job as > CREATE To > File**
- Linked servers, Database Mail profiles, credentials and proxies: right-click each > **Script ... as CREATE To**

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-07.png and uncomment the next line -->
<!-- ![Step 7: Script out the configuration and back up keys](../images/upgrade-sql-server-2019-to-2022/step-07.png) -->

> [!TIP]
> **Expected result:** The settings, trace flags, logins, jobs and linked server scripts are saved off the server, and the service master key and certificate backups are in the vault with their passwords.

## Step 8: Confirm a clean CHECKDB (the day before)

Do not upgrade a database that has corruption. Check when each database last passed DBCC CHECKDB:

```sql
SELECT name, DATABASEPROPERTYEX(name, 'LastGoodCheckDbTime') AS last_good_checkdb
FROM sys.databases
WHERE state_desc = N'ONLINE'
ORDER BY last_good_checkdb;
```

For any database older than your standard (for example 7 days), run it now:

```sql
DBCC CHECKDB (N'AppDB') WITH NO_INFOMSGS, ALL_ERRORMSGS;
```

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-08.png and uncomment the next line -->
<!-- ![Step 8: Confirm a clean CHECKDB](../images/upgrade-sql-server-2019-to-2022/step-08.png) -->

> [!TIP]
> **Expected result:** Every database, including master, model and msdb, has a last_good_checkdb within your standard, and any CHECKDB you ran finished with no errors.

## Step 9: Stop the application and take the final backups (change window starts)

1. Confirm with the application team that the application and its services are stopped
2. Check nothing is still connected (except you):

```sql
SELECT DB_NAME(database_id) AS database_name, login_name, host_name, program_name, COUNT(*) AS sessions
FROM sys.dm_exec_sessions
WHERE is_user_process = 1 AND session_id <> @@SPID
GROUP BY DB_NAME(database_id), login_name, host_name, program_name;
```

3. Take COPY_ONLY full backups of the system databases and every user database. COPY_ONLY does not disturb your normal backup chain.

```sql
BACKUP DATABASE master TO DISK = N'X:\SQLBackups\PreUpgrade\master.bak' WITH COPY_ONLY, CHECKSUM, COMPRESSION, INIT;
BACKUP DATABASE model  TO DISK = N'X:\SQLBackups\PreUpgrade\model.bak'  WITH COPY_ONLY, CHECKSUM, COMPRESSION, INIT;
BACKUP DATABASE msdb   TO DISK = N'X:\SQLBackups\PreUpgrade\msdb.bak'   WITH COPY_ONLY, CHECKSUM, COMPRESSION, INIT;

-- Each user database, or use the toolkit's backup-all-user-databases.sql
BACKUP DATABASE AppDB  TO DISK = N'X:\SQLBackups\PreUpgrade\AppDB.bak'  WITH COPY_ONLY, CHECKSUM, COMPRESSION, INIT;

-- Prove the files can be read
RESTORE VERIFYONLY FROM DISK = N'X:\SQLBackups\PreUpgrade\AppDB.bak' WITH CHECKSUM;
```

Toolkit: [backup-all-user-databases.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/backup-restore/backup-all-user-databases.sql)

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-09.png and uncomment the next line -->
<!-- ![Step 9: Stop the application and take the final backups](../images/upgrade-sql-server-2019-to-2022/step-09.png) -->

> [!TIP]
> **Expected result:** No application sessions are connected, every database has a fresh COPY_ONLY backup, and RESTORE VERIFYONLY reports *The backup set on file 1 is valid* for each one.

## Step 10: Take a VM snapshot

If SQL01 is a virtual machine, a snapshot is the fastest rollback. Stop the SQL Server services first so the snapshot is consistent:

```powershell
# Stops the Engine and everything that depends on it (Agent included)
Stop-Service -Name MSSQLSERVER -Force
Get-Service | Where-Object { $_.Name -like 'MSSQL*' -or $_.Name -like 'SQL*' } | Select-Object Name, Status
```

Ask the virtualization team to take the snapshot and confirm its name in the change. Leave the services stopped; Setup does not need them running.

A snapshot is not a backup: it lives on the same storage as the VM, and it slows the VM the longer it is kept. Agree when it will be deleted (Step 15).

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-10.png and uncomment the next line -->
<!-- ![Step 10: Take a VM snapshot](../images/upgrade-sql-server-2019-to-2022/step-10.png) -->

> [!TIP]
> **Expected result:** The virtualization team confirms the snapshot name and time, and the SQL services are stopped.

## Step 11: Run the upgrade in Setup (option A)

Start Setup from an elevated command prompt so it picks up the CU from Step 6:

```bat
cd /d C:\SQLInstall\SQL2022
setup.exe /ACTION=Upgrade /UPDATEENABLED=True /UPDATESOURCE="C:\SQLUpdates\2022"
```

Then in the wizard:

1. **Product Key**: confirm the edition matches today's edition (or the approved edition change)
2. **License Terms**: accept
3. **Product Updates**: the CU from `C:\SQLUpdates\2022` is listed. Continue
4. **Upgrade Rules**: everything must pass. A failed *pending restart* rule means restart the server and start again
5. **Azure extension for SQL Server**: if this page appears, clear it unless your Azure Arc onboarding process says to enable it
6. **Select Instance**: choose the instance (MSSQLSERVER for a default instance)
7. **Select Features**: shows the features installed today. You can't add features during an upgrade
8. **Instance Configuration**: keep the existing instance ID
9. **Server Configuration**: service accounts carry over. No change needed
10. **Full-text Upgrade**: choose **Import** (fastest). You repopulate the catalogs afterwards in Step 14
11. **Ready to Upgrade**: note the configuration file path shown at the bottom, then click **Upgrade**
12. **Complete**: every feature must show **Succeeded**. Restart the server if Setup asks

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-11.png and uncomment the next line -->
<!-- ![Step 11: Run the upgrade in Setup](../images/upgrade-sql-server-2019-to-2022/step-11.png) -->

> [!TIP]
> **Expected result:** The Complete page shows Succeeded for every feature.

## Step 12: Or run the upgrade from the command line (option B)

Same upgrade, unattended. Useful when you upgrade several servers the same way. Run from an elevated command prompt:

```bat
cd /d C:\SQLInstall\SQL2022
setup.exe /q /ACTION=Upgrade /INSTANCENAME=MSSQLSERVER /IACCEPTSQLSERVERLICENSETERMS ^
    /UPDATEENABLED=True /UPDATESOURCE="C:\SQLUpdates\2022" ^
    /FTUPGRADEOPTION=Import /INDICATEPROGRESS
:: Add /PID="<product key>" if your media does not have the key built in
echo Exit code: %ERRORLEVEL%
```

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-12.png and uncomment the next line -->
<!-- ![Step 12: Or run the upgrade from the command line](../images/upgrade-sql-server-2019-to-2022/step-12.png) -->

> [!TIP]
> **Expected result:** Exit code 0 (or 3010, which means success with a restart required).

## Step 13: Confirm the upgrade succeeded

Check the Setup log first, in PowerShell:

```powershell
Get-Content 'C:\Program Files\Microsoft SQL Server\160\Setup Bootstrap\Log\Summary.txt' -TotalCount 30
Get-Service | Where-Object { $_.Name -like 'MSSQL*' -or $_.Name -like 'SQL*' } |
    Select-Object Name, Status, StartType
```

Then connect with SSMS 20 or later and run:

```sql
SELECT  SERVERPROPERTY('ProductVersion')     AS product_version,    -- 16.0.x = SQL Server 2022
        SERVERPROPERTY('ProductUpdateLevel') AS cumulative_update,
        SERVERPROPERTY('Edition')            AS edition;

-- Every database ONLINE, compatibility level unchanged (still 150)
SELECT name, state_desc, compatibility_level FROM sys.databases ORDER BY name;

-- The upgrade scripts finished, and no errors since startup
EXEC sys.xp_readerrorlog 0, 1, N'upgrade';
EXEC sys.xp_readerrorlog 0, 1, N'error';
```

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-13.png and uncomment the next line -->
<!-- ![Step 13: Confirm the upgrade succeeded](../images/upgrade-sql-server-2019-to-2022/step-13.png) -->

> [!TIP]
> **Expected result:** Summary.txt shows *Final result: Passed*, the Engine and Agent services are Running, ProductVersion starts with 16.0 at the CU you planned, every database is ONLINE at its old compatibility level, and the error log has no unexplained errors.

## Step 14: Post-upgrade tasks

Work through these before the application reconnects.

**Update statistics** in every user database:

```sql
DECLARE @sql nvarchar(max);
SELECT @sql = STRING_AGG(CAST(N'EXEC ' + QUOTENAME(name) + N'.sys.sp_updatestats;' AS nvarchar(max)), CHAR(10))
FROM sys.databases
WHERE database_id > 4 AND state_desc = N'ONLINE' AND is_read_only = 0;
IF @sql IS NOT NULL EXEC sys.sp_executesql @sql;
```

**Test every linked server.** Native Client providers and certificate checks are the usual problems (see the [linked server runbook](../provision/linked-server.md), Step 8):

```sql
DECLARE @name sysname;
DECLARE c CURSOR LOCAL FAST_FORWARD FOR SELECT name FROM sys.servers WHERE is_linked = 1;
OPEN c;
FETCH NEXT FROM c INTO @name;
WHILE @@FETCH_STATUS = 0
BEGIN
    BEGIN TRY
        EXEC master.dbo.sp_testlinkedserver @name;
        PRINT N'OK    ' + @name;
    END TRY
    BEGIN CATCH
        PRINT N'FAIL  ' + @name + N': ' + ERROR_MESSAGE();
    END CATCH;
    FETCH NEXT FROM c INTO @name;
END;
CLOSE c;
DEALLOCATE c;
```

**And the rest:**

- **Full-text**: repopulate any full-text indexes (in SSMS, right-click the full-text catalog > **Rebuild**, or `ALTER FULLTEXT INDEX ON dbo.YourTable START FULL POPULATION;`)
- **SSISDB** (if Step 3f found it): in SSMS expand **Integration Services Catalogs**, right-click **SSISDB** > **Database Upgrade**, and follow the wizard
- **Database Mail**: send a test with [database-mail-check.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/monitoring/database-mail-check.sql)
- **Agent jobs**: run one low-risk job by hand and confirm it succeeds
- **Machine Learning Services** (if Step 3b found it): install the R or Python runtime the application needs and re-test
- **New full backups**: take a normal full backup of every database so your backup chain starts on SQL Server 2022
- **Monitoring**: confirm your monitoring tools see the instance as SQL Server 2022 and alerts are flowing
- **Health checks**: run the [new instance validation runbook](../validate/new-instance-validation.md)

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-14.png and uncomment the next line -->
<!-- ![Step 14: Post-upgrade tasks](../images/upgrade-sql-server-2019-to-2022/step-14.png) -->

> [!TIP]
> **Expected result:** Statistics updated, every linked server prints OK, the test email arrives, the test job succeeds, a new full backup exists for every database, and monitoring shows the new version.

## Step 15: Hand back and monitor

1. Hand back to the application team for their smoke test. Once they write live data to SQL01, you have passed the point of no return (Step 17)
2. Update the server's configuration item in your CMDB with the new version and CU
3. Watch closely for the next few days: failed jobs ([failed-jobs.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/agent-jobs/failed-jobs.sql)), CPU and waits compared with the Step 5 baseline, and the Query Store **Regressed Queries** report
4. Delete the VM snapshot on the date agreed in Step 10
5. Close the change with the evidence from Steps 1, 3, 9, 13 and 14 attached

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-15.png and uncomment the next line -->
<!-- ![Step 15: Hand back and monitor](../images/upgrade-sql-server-2019-to-2022/step-15.png) -->

> [!TIP]
> **Expected result:** The application team signs off, there are no new failed jobs or performance regressions, and the snapshot has been deleted.

## Step 16: Raise the compatibility level to 160 (a later change)

Do this as its own change, one database at a time, after the upgrade has been stable for a while and Query Store has captured a baseline at level 150. Level 160 turns on the newer SQL Server 2022 optimizer features, such as Parameter Sensitive Plan optimization.

```sql
ALTER DATABASE [AppDB] SET COMPATIBILITY_LEVEL = 160;

-- If an important query gets slower: in SSMS open Query Store > Regressed Queries,
-- pick the query and force its last good plan. If the problem is widespread, go back:
ALTER DATABASE [AppDB] SET COMPATIBILITY_LEVEL = 150;
```

SSMS also has the **Query Tuning Assistant** (right-click the database > **Tasks** > **Database Upgrade** > **New Database Upgrade Session**), which runs this comparison for you.

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-16.png and uncomment the next line -->
<!-- ![Step 16: Raise the compatibility level to 160](../images/upgrade-sql-server-2019-to-2022/step-16.png) -->

> [!TIP]
> **Expected result:** The database runs at level 160 with no regressions in the Query Store reports, or it has been set back to 150 with the reason recorded.

## Step 17: Roll back if needed

Decide quickly. Every option below loses anything written to SQL01 after the Step 9 backups, which is why the point of no return matters.

**Option 1: revert the VM snapshot (fastest).** Ask the virtualization team to revert to the Step 10 snapshot. SQL01 comes back as SQL Server 2019 exactly as it was before Setup ran. Start the services and check `@@VERSION`.

**Option 2: rebuild on SQL Server 2019 from backups (no snapshot).**

1. Uninstall SQL Server 2022 from SQL01
2. Install SQL Server 2019 with the same instance name, collation, file paths and service accounts, and patch it to the **same CU as before** (from Step 1)
3. Restore master in single-user mode:

```bat
net stop MSSQLSERVER /y
net start MSSQLSERVER /mSQLCMD
sqlcmd -S . -E -Q "RESTORE DATABASE master FROM DISK = N'X:\SQLBackups\PreUpgrade\master.bak' WITH REPLACE"
:: SQL Server stops by itself after master is restored
net start MSSQLSERVER
```

4. With SQL Server Agent stopped, restore msdb and model, then every user database:

```sql
RESTORE DATABASE msdb  FROM DISK = N'X:\SQLBackups\PreUpgrade\msdb.bak'  WITH REPLACE;
RESTORE DATABASE model FROM DISK = N'X:\SQLBackups\PreUpgrade\model.bak' WITH REPLACE;
RESTORE DATABASE AppDB FROM DISK = N'X:\SQLBackups\PreUpgrade\AppDB.bak' WITH REPLACE, RECOVERY;
```

5. Restore the service master key and certificates from Step 7 if TDE databases fail to open, then start Agent and run Step 13's checks (expecting 15.0.x)

<!-- Screenshot: save as docs/images/upgrade-sql-server-2019-to-2022/step-17.png and uncomment the next line -->
<!-- ![Step 17: Roll back if needed](../images/upgrade-sql-server-2019-to-2022/step-17.png) -->

> [!TIP]
> **Expected result:** SQL01 reports 15.0.x at the original CU, every database is ONLINE, and the application team confirms the application works.

## Validation sign-off

- [ ] Instance confirmed standalone, OS Windows Server 2016 or later, .NET 4.7.2 or later, no pending restart
- [ ] Every Step 3 finding has an owner and a plan (Native Client linked servers, ML Services, Stretch, PolyBase Hadoop, replication, log shipping)
- [ ] Application, monitoring, backup and ETL tools confirmed for SQL Server 2022
- [ ] Query Store on and a baseline saved before the upgrade
- [ ] Configuration, logins, jobs and linked servers scripted out and stored off the server
- [ ] Service master key and certificates backed up to the vault
- [ ] Clean CHECKDB for every database
- [ ] COPY_ONLY full backups of every database, verified
- [ ] VM snapshot taken (or Option 2 rollback accepted in the change)
- [ ] Summary.txt shows Passed; ProductVersion 16.0.x at the planned CU
- [ ] Every database ONLINE at its original compatibility level
- [ ] Statistics updated, linked servers, Database Mail, Agent and SSISDB tested
- [ ] New full backups taken on SQL Server 2022
- [ ] Application team sign-off; snapshot deleted on the agreed date

## Sources

- **Microsoft Learn:** [Supported version and edition upgrades (SQL Server 2022)](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/supported-version-and-edition-upgrades-2022)
- **Microsoft Learn:** [Hardware and software requirements for SQL Server 2022](https://learn.microsoft.com/en-us/sql/sql-server/install/hardware-and-software-requirements-for-installing-sql-server-2022)
- **Microsoft Learn:** [Upgrade SQL Server](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/upgrade-sql-server)
- **Microsoft Learn:** [Upgrade SQL Server using the Installation Wizard (Setup)](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/upgrade-sql-server-using-the-installation-wizard-setup)
- **Microsoft Learn:** [Install SQL Server from the command prompt](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/install-sql-server-from-the-command-prompt)
- **Microsoft Learn:** [Complete the Database Engine upgrade](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/complete-the-database-engine-upgrade)
- **Microsoft Learn:** [Change the database compatibility level and use the Query Store](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/change-the-database-compatibility-mode-and-use-the-query-store)
- **Microsoft Learn:** [Discontinued Database Engine functionality in SQL Server](https://learn.microsoft.com/en-us/sql/database-engine/discontinued-database-engine-functionality-in-sql-server)
- **MSSQLTips:** [SQL Server upgrades and migrations tips](https://www.mssqltips.com/sql-server-tip-category/24/upgrades-and-migrations/)
- **SQLShack:** [Database upgrade using the Query Tuning Assistant wizard in SSMS](https://www.sqlshack.com/database-upgrade-using-the-query-tuning-assistant-wizard-in-ssms-18/)
- **SQLShack:** [Different ways to identify and change compatibility levels in SQL Server](https://www.sqlshack.com/different-ways-to-identify-and-change-compatibility-levels-in-sql-server/)

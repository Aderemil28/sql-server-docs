# New instance validation and sign-off

**Status:** Draft · **Applies to:** SQL Server 2016 to 2022 · **Steps:** 9

Prove a newly built instance is ready for an application before you hand it over: services, patch level, configuration, storage, connectivity, backups and alerting. Run it after the install and configuration runbooks, and again after any rebuild.

> [!NOTE]
> **Before you start**
>
> - Install and configuration runbooks completed
> - Access to the MySqlScripts toolkit (linked in Sources)
> - A test database you can back up and restore

## Step 1: Services and patch level

Confirm what is running and at what build.

```sql
SELECT @@SERVERNAME AS server_name,
       SERVERPROPERTY('ProductVersion')     AS product_version,
       SERVERPROPERTY('ProductUpdateLevel') AS cumulative_update;

SELECT servicename, service_account, startup_type_desc, status_desc
FROM sys.dm_server_services;
```

<!-- Screenshot: save as docs/images/new-instance-validation/step-01.png and uncomment the next line -->
<!-- ![Step 1: Services and patch level](../images/new-instance-validation/step-01.png) -->

> [!TIP]
> **Expected result:** The build matches the current approved CU; Engine and Agent are Running and Automatic.

## Step 2: Configuration matches the baseline

Compare the key settings with the configuration runbook.

```sql
SELECT name, value_in_use
FROM sys.configurations
WHERE name IN (N'max server memory (MB)', N'max degree of parallelism', N'cost threshold for parallelism',
               N'optimize for ad hoc workloads', N'backup compression default', N'backup checksum default',
               N'remote admin connections')
ORDER BY name;
```

<!-- Screenshot: save as docs/images/new-instance-validation/step-02.png and uncomment the next line -->
<!-- ![Step 2: Configuration matches the baseline](../images/new-instance-validation/step-02.png) -->

> [!TIP]
> **Expected result:** Every value matches the baseline.

## Step 3: Storage and file layout

Run [disk-space-by-volume.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/storage/disk-space-by-volume.sql) and [database-file-sizes-and-free-space.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/storage/database-file-sizes-and-free-space.sql) from the toolkit. Check every database file sits on the right volume and nothing is on C:.

<!-- Screenshot: save as docs/images/new-instance-validation/step-03.png and uncomment the next line -->
<!-- ![Step 3: Storage and file layout](../images/new-instance-validation/step-03.png) -->

> [!TIP]
> **Expected result:** Data, log, tempdb and backup files are on their own volumes with healthy free space.

## Step 4: tempdb and IFI

```sql
SELECT COUNT(*) AS data_files, MIN(size) / 128 AS min_mb, MAX(size) / 128 AS max_mb
FROM tempdb.sys.database_files WHERE type = 0;

SELECT servicename, instant_file_initialization_enabled
FROM sys.dm_server_services WHERE servicename LIKE N'SQL Server (%';
```

<!-- Screenshot: save as docs/images/new-instance-validation/step-04.png and uncomment the next line -->
<!-- ![Step 4: tempdb and IFI](../images/new-instance-validation/step-04.png) -->

> [!TIP]
> **Expected result:** min_mb equals max_mb and IFI is Y.

## Step 5: Connectivity from the application side

From the application or jump server, run [Test-SqlConnectivity.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/powershell/Test-SqlConnectivity.ps1) from the toolkit. It checks DNS, the TCP port on every resolved IP and a real login.

```powershell
.\Test-SqlConnectivity.ps1 -HostName <server FQDN> -Port 1433 -TestLogin
```

<!-- Screenshot: save as docs/images/new-instance-validation/step-05.png and uncomment the next line -->
<!-- ![Step 5: Connectivity from the application side](../images/new-instance-validation/step-05.png) -->

> [!TIP]
> **Expected result:** DNS resolves, the port is OPEN and the login test connects with auth=KERBEROS for domain accounts.

## Step 6: Backup and restore test

Take a backup of a test database and restore it under a new name. A backup is only proven once it has been restored.

```sql
BACKUP DATABASE [TestDB] TO DISK = N'X:\SQLBackups\TestDB_validation.bak' WITH COPY_ONLY, COMPRESSION, CHECKSUM, STATS = 10;

RESTORE VERIFYONLY FROM DISK = N'X:\SQLBackups\TestDB_validation.bak' WITH CHECKSUM;

RESTORE DATABASE [TestDB_restore_check] FROM DISK = N'X:\SQLBackups\TestDB_validation.bak'
WITH MOVE N'TestDB' TO N'D:\SQLData\TestDB_restore_check.mdf',
     MOVE N'TestDB_log' TO N'L:\SQLLogs\TestDB_restore_check_log.ldf', RECOVERY, STATS = 10;

DBCC CHECKDB (N'TestDB_restore_check') WITH NO_INFOMSGS;
DROP DATABASE [TestDB_restore_check];
```

<!-- Screenshot: save as docs/images/new-instance-validation/step-06.png and uncomment the next line -->
<!-- ![Step 6: Backup and restore test](../images/new-instance-validation/step-06.png) -->

> [!TIP]
> **Expected result:** The backup, verify and restore complete, and CHECKDB returns no errors. Logical file names come from RESTORE FILELISTONLY if yours differ.

## Step 7: Maintenance jobs and alerts

Run [jobs-schedule-and-next-run.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/agent-jobs/jobs-schedule-and-next-run.sql) from the toolkit. Confirm backup, integrity check and index maintenance jobs exist, are enabled and have a next run time. Raise a test alert to prove email works.

```sql
RAISERROR (N'Validation test alert, please ignore', 17, 1) WITH LOG;
```

<!-- Screenshot: save as docs/images/new-instance-validation/step-07.png and uncomment the next line -->
<!-- ![Step 7: Maintenance jobs and alerts](../images/new-instance-validation/step-07.png) -->

> [!TIP]
> **Expected result:** Jobs are scheduled, and the test alert email arrives in the team mailbox.

## Step 8: Error log review

Look for anything unexpected since the last restart: failed logins, I/O warnings, memory messages.

```sql
EXEC master.dbo.xp_readerrorlog 0, 1, N'error';
EXEC master.dbo.xp_readerrorlog 0, 1, N'warning';
```

<!-- Screenshot: save as docs/images/new-instance-validation/step-08.png and uncomment the next line -->
<!-- ![Step 8: Error log review](../images/new-instance-validation/step-08.png) -->

> [!TIP]
> **Expected result:** No unexplained errors. Anything found is noted and resolved before sign-off.

## Step 9: Sign off

Tick every item in the sign-off list below, attach the outputs to the change record and hand over to the application team.

<!-- Screenshot: save as docs/images/new-instance-validation/step-09.png and uncomment the next line -->
<!-- ![Step 9: Sign off](../images/new-instance-validation/step-09.png) -->

## Validation sign-off

- [ ] Build matches the approved CU
- [ ] Services running under the right accounts, Automatic
- [ ] Configuration matches the baseline
- [ ] Files on the correct volumes, nothing on C:
- [ ] tempdb files equal and IFI enabled
- [ ] Application server can connect (DNS, port, login)
- [ ] Test backup restored and CHECKDB clean
- [ ] Backup, CHECKDB and index jobs scheduled
- [ ] Test alert email received
- [ ] Error log reviewed, nothing outstanding

## Sources

- **MSSQLTips:** [SQL Server Post SQL Installation Checklist (Part 4 of 4)](https://www.mssqltips.com/sqlservertip/2284/sql-server-post-sql-installation-checklist-part-4-of-4/)
- **MSSQLTips:** [SQL Server Restart Validation Checklist](https://www.mssqltips.com/sqlservertip/2070/sql-server-restart-validation-checklist/)
- **Microsoft Learn:** [tempdb database](https://learn.microsoft.com/en-us/sql/relational-databases/databases/tempdb-database)
- **Toolkit:** [MySqlScripts toolkit on GitHub (scripts referenced in this runbook)](https://github.com/Aderemil28/MySqlScripts)

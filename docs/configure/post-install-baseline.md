# Post-install configuration baseline

**Status:** Draft · **Applies to:** SQL Server 2016 to 2022 · **Steps:** 10

Apply the standard instance settings straight after install, before any database or application lands on the server: memory, parallelism, tempdb, backup defaults, error logs and alerts.

> [!NOTE]
> **Before you start**
>
> - Instance installed and patched (see Install SQL Server 2022)
> - sysadmin access
> - The server's core count and total RAM noted
> - Database Mail profile created if alerts should email (see [database-mail-check.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/monitoring/database-mail-check.sql))

## Step 1: Capture the current settings

Save the before state so you can show what changed in the change record.

```sql
SELECT name, value, value_in_use, description
FROM sys.configurations
ORDER BY name;
```

<!-- Screenshot: save as docs/images/post-install-baseline/step-01.png and uncomment the next line -->
<!-- ![Step 1: Capture the current settings](../images/post-install-baseline/step-01.png) -->

> [!TIP]
> **Expected result:** A full list of settings, saved as a CSV or screenshot.

## Step 2: Set max server memory

Leave enough memory for Windows and anything else on the box. A common starting point on a dedicated server: leave 4 GB, plus 1 GB for every 8 GB above 16 GB. Adjust later from monitoring. This script works out the value and prints the command; it does not change anything.

```sql
DECLARE @total_mb int = (SELECT total_physical_memory_kb / 1024 FROM sys.dm_os_sys_memory);
DECLARE @os_mb    int = CASE WHEN @total_mb <= 16384 THEN 4096
                             ELSE 4096 + ((@total_mb - 16384) / 8192) * 1024 END;

SELECT @total_mb AS total_mb, @os_mb AS reserve_for_os_mb, @total_mb - @os_mb AS suggested_max_server_memory_mb;

PRINT 'EXEC sp_configure ''show advanced options'', 1; RECONFIGURE;';
PRINT 'EXEC sp_configure ''max server memory (MB)'', ' + CAST(@total_mb - @os_mb AS varchar(10)) + '; RECONFIGURE;';
```

<!-- Screenshot: save as docs/images/post-install-baseline/step-02.png and uncomment the next line -->
<!-- ![Step 2: Set max server memory](../images/post-install-baseline/step-02.png) -->

> [!TIP]
> **Expected result:** A suggested value well below total RAM. Run the printed command once you agree with it.

## Step 3: Set MAXDOP and cost threshold for parallelism

Microsoft's guidance: on a single NUMA node with 8 or fewer cores, MAXDOP = number of cores; with more than 8, MAXDOP = 8. Raise **cost threshold for parallelism** from the default of 5 to around 50 so small queries do not go parallel.

```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'max degree of parallelism', 8;        -- adjust to the guidance above
EXEC sp_configure 'cost threshold for parallelism', 50;
RECONFIGURE;
```

<!-- Screenshot: save as docs/images/post-install-baseline/step-03.png and uncomment the next line -->
<!-- ![Step 3: Set MAXDOP and cost threshold for parallelism](../images/post-install-baseline/step-03.png) -->

> [!TIP]
> **Expected result:** value_in_use shows the new values in sys.configurations.

## Step 4: Turn on the low-risk defaults

- **optimize for ad hoc workloads** keeps single-use plans small in the plan cache
- **backup compression default** compresses every backup unless told otherwise
- **backup checksum default** validates pages as they are backed up
- **remote admin connections** lets you use the dedicated admin connection (DAC) from another machine when the server is in trouble

```sql
EXEC sp_configure 'optimize for ad hoc workloads', 1;
EXEC sp_configure 'backup compression default', 1;
EXEC sp_configure 'backup checksum default', 1;
EXEC sp_configure 'remote admin connections', 1;
RECONFIGURE;
```

<!-- Screenshot: save as docs/images/post-install-baseline/step-04.png and uncomment the next line -->
<!-- ![Step 4: Turn on the low-risk defaults](../images/post-install-baseline/step-04.png) -->

> [!TIP]
> **Expected result:** All four show value_in_use = 1.

## Step 5: Check tempdb

Setup normally creates tempdb correctly. Confirm the data files are equal in size and growth. Unequal files defeat the round-robin allocation that multiple files are for.

```sql
SELECT name, type_desc, physical_name, size / 128 AS size_mb,
       CASE WHEN is_percent_growth = 1 THEN CAST(growth AS varchar(10)) + ' %'
            ELSE CAST(growth / 128 AS varchar(10)) + ' MB' END AS growth
FROM tempdb.sys.database_files;
```

<!-- Screenshot: save as docs/images/post-install-baseline/step-05.png and uncomment the next line -->
<!-- ![Step 5: Check tempdb](../images/post-install-baseline/step-05.png) -->

> [!TIP]
> **Expected result:** Every ROWS file has the same size and the same fixed-MB growth.

## Step 6: Set sensible defaults on model

New databases copy model. Set fixed-MB growth so new databases do not grow by percent. Set the recovery model your standard expects (FULL if new databases need point-in-time restore).

```sql
ALTER DATABASE model MODIFY FILE (NAME = N'modeldev', SIZE = 256MB, FILEGROWTH = 256MB);
ALTER DATABASE model MODIFY FILE (NAME = N'modellog', SIZE = 256MB, FILEGROWTH = 256MB);
ALTER DATABASE model SET RECOVERY FULL;
```

<!-- Screenshot: save as docs/images/post-install-baseline/step-06.png and uncomment the next line -->
<!-- ![Step 6: Set sensible defaults on model](../images/post-install-baseline/step-06.png) -->

> [!TIP]
> **Expected result:** SELECT name, recovery_model_desc FROM sys.databases WHERE name = 'model' shows the recovery model you set.

## Step 7: Keep more error logs

The default of 6 error logs is often only a few days on a busy server. Keep 30, and cycle the log nightly with an Agent job (`EXEC sp_cycle_errorlog;`) so each file covers one day.

```sql
EXEC master.dbo.xp_instance_regwrite
     N'HKEY_LOCAL_MACHINE',
     N'Software\Microsoft\MSSQLServer\MSSQLServer',
     N'NumErrorLogs', REG_DWORD, 30;
```

<!-- Screenshot: save as docs/images/post-install-baseline/step-07.png and uncomment the next line -->
<!-- ![Step 7: Keep more error logs](../images/post-install-baseline/step-07.png) -->

> [!TIP]
> **Expected result:** In SSMS, Management > SQL Server Logs > Configure shows 30.

## Step 8: Create an operator and severity alerts

Alert on severity 17 to 25 and on errors 823, 824 and 825 (I/O and corruption warnings). Replace the email address with your DBA team's mailbox.

```sql
USE msdb;
EXEC dbo.sp_add_operator @name = N'DBA Team', @enabled = 1, @email_address = N'dba-team@example.com';

DECLARE @sev int = 17, @name sysname;
WHILE @sev <= 25
BEGIN
    SET @name = N'Severity ' + CAST(@sev AS nvarchar(3));
    EXEC dbo.sp_add_alert @name = @name, @severity = @sev, @include_event_description_in = 1, @delay_between_responses = 300;
    EXEC dbo.sp_add_notification @alert_name = @name, @operator_name = N'DBA Team', @notification_method = 1;
    SET @sev += 1;
END;

DECLARE @err int, @i int = 1;
WHILE @i <= 3
BEGIN
    SET @err = CHOOSE(@i, 823, 824, 825);
    SET @name = N'Error ' + CAST(@err AS nvarchar(5));
    EXEC dbo.sp_add_alert @name = @name, @message_id = @err, @include_event_description_in = 1, @delay_between_responses = 300;
    EXEC dbo.sp_add_notification @alert_name = @name, @operator_name = N'DBA Team', @notification_method = 1;
    SET @i += 1;
END;
```

<!-- Screenshot: save as docs/images/post-install-baseline/step-08.png and uncomment the next line -->
<!-- ![Step 8: Create an operator and severity alerts](../images/post-install-baseline/step-08.png) -->

> [!TIP]
> **Expected result:** SQL Server Agent > Alerts lists Severity 17 to 25 and Error 823, 824, 825, each notifying DBA Team.

## Step 9: Point Agent at Database Mail

In SSMS, right-click **SQL Server Agent** > **Properties** > **Alert System**. Tick **Enable mail profile**, pick the profile, then restart SQL Server Agent. Without this, alerts fire but no email is sent.

<!-- Screenshot: save as docs/images/post-install-baseline/step-09.png and uncomment the next line -->
<!-- ![Step 9: Point Agent at Database Mail](../images/post-install-baseline/step-09.png) -->

> [!TIP]
> **Expected result:** A test alert (RAISERROR ('test', 17, 1) WITH LOG) arrives in the team mailbox.

## Step 10: Record the after state

Re-run step 1 and save the output next to the before state in the change record.

<!-- Screenshot: save as docs/images/post-install-baseline/step-10.png and uncomment the next line -->
<!-- ![Step 10: Record the after state](../images/post-install-baseline/step-10.png) -->

## Validation sign-off

- [ ] max server memory is set below total RAM, with headroom for the OS
- [ ] MAXDOP and cost threshold for parallelism match the standard
- [ ] optimize for ad hoc workloads, backup compression and backup checksum defaults are on
- [ ] remote admin connections (DAC) is on
- [ ] tempdb data files are equal in size and growth
- [ ] model uses fixed-MB growth and the standard recovery model
- [ ] Error log count is 30 with a nightly cycle job
- [ ] Severity 17 to 25 and 823/824/825 alerts notify the DBA operator, and a test alert arrived

## Sources

- **Microsoft Learn:** [Server memory configuration options](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/server-memory-server-configuration-options)
- **Microsoft Learn:** [Configure the max degree of parallelism (MAXDOP)](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/configure-the-max-degree-of-parallelism-server-configuration-option)
- **Microsoft Learn:** [tempdb database](https://learn.microsoft.com/en-us/sql/relational-databases/databases/tempdb-database)
- **SQLShack:** [Best practices after installing Microsoft SQL Server](https://www.sqlshack.com/best-practices-after-installing-microsoft-sql-server/)
- **MSSQLTips:** [SQL Server Post SQL Installation Checklist (Part 4 of 4)](https://www.mssqltips.com/sqlservertip/2284/sql-server-post-sql-installation-checklist-part-4-of-4/)

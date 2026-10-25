# Refresh a non-production database from production

**Status:** Draft · **Applies to:** SQL Server 2016 to 2022 · **Steps:** 15

Overwrite a test or development copy of a database with a recent production backup, then put the target environment's own users, permissions and settings back, mask sensitive data, and prove the refresh worked before handing it to the requester.

In this runbook **PRDSQL01** is the production (source) server, **TSTSQL01** is the non-production (target) server, and **AppDB** is the database being refreshed. **\\\\backupshare\\SQLRefresh** is a file share both servers can reach, and **X:\\SQLRefresh** is a local folder on TSTSQL01. Replace these with your own names.

> [!NOTE]
> **Before you start**
>
> - An approved request naming the source database, the target, and the point in time wanted (latest, or a specific date and time)
> - Data owner approval to copy production data to the target environment, and the approved **masking script** if the data includes customer or other sensitive information
> - TSTSQL01 runs the **same or a newer** SQL Server version than PRDSQL01. A backup can't be restored to an older version
> - sysadmin on TSTSQL01, and rights to take a COPY_ONLY backup on PRDSQL01 (or access to an existing production backup)
> - An agreed outage on the target database: everyone connected to it is disconnected during the refresh
> - If AppDB on TSTSQL01 is in an availability group, it has to be removed from the AG before the restore and added back afterwards. Plan that with this runbook

> [!WARNING]
> The backup file contains **unmasked production data** until Step 11 is done. Keep it on approved storage only, and delete every copy in Step 14.

## Step 1: Confirm the request

Fill this in from the ticket before touching anything. If any line is blank, go back to the requester.

| Item | Answer |
|---|---|
| Source server and database | PRDSQL01 / AppDB |
| Target server and database | TSTSQL01 / AppDB |
| Point in time | Latest full backup, or a specific date and time |
| Masking required | Yes / No, and the approved script name |
| What to keep on the target | Target users and permissions (normally yes), anything else the team asked for |
| Outage window on the target | Date, time, who has been told |
| Requester who will test and sign off | Name |

<!-- Screenshot: save as docs/images/database-refresh/step-01.png and uncomment the next line -->
<!-- ![Step 1: Confirm the request](../images/database-refresh/step-01.png) -->

> [!TIP]
> **Expected result:** Every row has an answer, and the approvals are attached to the ticket.

## Step 2: Check there is enough space on the target

The target needs room for the **allocated size** of the source database files, not the backup size. Run on PRDSQL01:

```sql
-- Run on PRDSQL01: allocated size of each file in the source database
SELECT  DB_NAME(database_id) AS database_name, type_desc, name AS logical_name,
        CAST(size * 8.0 / 1024 AS decimal(18,1)) AS size_mb
FROM sys.master_files
WHERE database_id = DB_ID(N'AppDB');
```

Then run on TSTSQL01:

```sql
-- Run on TSTSQL01: current files of the target database, and free space on each drive
SELECT  mf.name AS logical_name, mf.type_desc, mf.physical_name,
        CAST(mf.size * 8.0 / 1024 AS decimal(18,1))                  AS size_mb,
        vs.volume_mount_point                                         AS drive,
        CAST(vs.available_bytes / 1048576.0 AS decimal(18,1))         AS drive_free_mb
FROM sys.master_files AS mf
CROSS APPLY sys.dm_os_volume_stats(mf.database_id, mf.file_id) AS vs
WHERE mf.database_id = DB_ID(N'AppDB');
```

The restore overwrites the target's existing AppDB files, so the space they use today counts as available. If AppDB is new on TSTSQL01, use [disk-space-by-volume.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/storage/disk-space-by-volume.sql) instead. You also need room for the backup file in X:\\SQLRefresh if you copy it locally (Step 6).

<!-- Screenshot: save as docs/images/database-refresh/step-02.png and uncomment the next line -->
<!-- ![Step 2: Check there is enough space on the target](../images/database-refresh/step-02.png) -->

> [!TIP]
> **Expected result:** For each drive, drive_free_mb plus the size of the target's current files on that drive is larger than the source files that will land there.

## Step 3: On the target, save what you need to put back

The restore replaces everything in the target database with production's version, including users and permissions. Save the target's own setup first. Run on **TSTSQL01**:

```sql
-- Run on TSTSQL01: settings to restore after the refresh
SELECT  name, SUSER_SNAME(owner_sid) AS owner_name, recovery_model_desc, compatibility_level,
        is_trustworthy_on, is_broker_enabled, is_query_store_on,
        snapshot_isolation_state_desc, is_read_committed_snapshot_on
FROM sys.databases
WHERE name = N'AppDB';

-- File layout: you need these paths for the MOVE clauses in Step 8
SELECT name AS logical_name, type_desc, physical_name
FROM sys.master_files
WHERE database_id = DB_ID(N'AppDB');
```

Then:

1. Run [script-out-database-users-and-permissions.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/security/script-out-database-users-and-permissions.sql) in AppDB on TSTSQL01 and save the output as `AppDB_TST_permissions.sql`. This is how the target's access comes back in Step 10
2. Ask the requester whether any **target-only data** must survive (for example test configuration tables). If so, export those tables now (BCP, SSIS or `SELECT INTO` a DBA database) and plan to load them back after Step 11

<!-- Screenshot: save as docs/images/database-refresh/step-03.png and uncomment the next line -->
<!-- ![Step 3: On the target, save what you need to put back](../images/database-refresh/step-03.png) -->

> [!TIP]
> **Expected result:** The settings, file paths and AppDB_TST_permissions.sql are saved with the ticket.

## Step 4: On the source, take a COPY_ONLY backup

Use COPY_ONLY so the refresh backup doesn't affect production's differential backups. Run on **PRDSQL01**. The SQL Server service account on PRDSQL01 needs write access to the share.

```sql
-- Run on PRDSQL01
BACKUP DATABASE AppDB
TO DISK = N'\\backupshare\SQLRefresh\AppDB_FULL_20261024.bak'
WITH COPY_ONLY, CHECKSUM, COMPRESSION, INIT, STATS = 10;
```

If you are using an existing production backup instead, find the latest one:

```sql
-- Run on PRDSQL01: latest full backups of AppDB
SELECT TOP (5) bs.backup_finish_date, bs.is_copy_only, bs.has_backup_checksums,
       CAST(bs.compressed_backup_size / 1048576.0 AS decimal(18,1)) AS size_mb,
       bmf.physical_device_name
FROM msdb.dbo.backupset AS bs
JOIN msdb.dbo.backupmediafamily AS bmf ON bmf.media_set_id = bs.media_set_id
WHERE bs.database_name = N'AppDB' AND bs.type = 'D'
ORDER BY bs.backup_finish_date DESC;
```

For a specific point in time you need the full backup plus the differential and log backups up to that time. [generate-backup-and-restore-commands.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/backup-restore/generate-backup-and-restore-commands.sql) builds that restore sequence.

<!-- Screenshot: save as docs/images/database-refresh/step-04.png and uncomment the next line -->
<!-- ![Step 4: On the source, take a COPY_ONLY backup](../images/database-refresh/step-04.png) -->

> [!TIP]
> **Expected result:** The backup finishes with *BACKUP DATABASE successfully processed*, and the file is on the share.

## Step 5: If the source uses TDE, check the target has the certificate

A TDE-encrypted backup can only be restored on a server that has the same certificate. Run on **PRDSQL01**:

```sql
-- Run on PRDSQL01: is AppDB encrypted, and with which certificate?
SELECT  DB_NAME(k.database_id) AS database_name, k.encryption_state,   -- 3 = encrypted
        c.name AS certificate_name, c.thumbprint
FROM sys.dm_database_encryption_keys AS k
JOIN master.sys.certificates AS c ON c.thumbprint = k.encryptor_thumbprint
WHERE k.database_id = DB_ID(N'AppDB');
```

If it returns a row, check TSTSQL01 for the same thumbprint:

```sql
-- Run on TSTSQL01: paste the thumbprint from PRDSQL01
SELECT name, thumbprint, expiry_date
FROM master.sys.certificates
WHERE thumbprint = 0x0000000000000000000000000000000000000000;
```

If the certificate isn't on TSTSQL01, putting a production TDE certificate on a non-production server needs **security approval**. Many organizations don't allow it, and instead restore and mask in a secured staging environment first. With approval, back up the certificate on PRDSQL01 and create it on TSTSQL01:

```sql
-- On PRDSQL01
BACKUP CERTIFICATE TDECert TO FILE = N'\\backupshare\SQLRefresh\TDECert.cer'
    WITH PRIVATE KEY (FILE = N'\\backupshare\SQLRefresh\TDECert.pvk',
                      ENCRYPTION BY PASSWORD = N'<StrongPassword>');

-- On TSTSQL01 (needs a database master key in master first)
USE master;
CREATE CERTIFICATE TDECert FROM FILE = N'\\backupshare\SQLRefresh\TDECert.cer'
    WITH PRIVATE KEY (FILE = N'\\backupshare\SQLRefresh\TDECert.pvk',
                      DECRYPTION BY PASSWORD = N'<StrongPassword>');
```

Delete the .cer and .pvk files from the share as soon as the certificate is created. The same applies if the backup used `WITH ENCRYPTION`.

<!-- Screenshot: save as docs/images/database-refresh/step-05.png and uncomment the next line -->
<!-- ![Step 5: If the source uses TDE, check the target has the certificate](../images/database-refresh/step-05.png) -->

> [!TIP]
> **Expected result:** AppDB is not encrypted, or TSTSQL01 has a certificate with the same thumbprint.

## Step 6: Copy the backup to the target and check it

You can restore straight from the share, but a local copy is faster and isn't affected by network drops during the restore. Run on **TSTSQL01** in an elevated command prompt:

```bat
robocopy \\backupshare\SQLRefresh X:\SQLRefresh AppDB_FULL_20261024.bak /J /R:3 /W:10 /NP /LOG:C:\Temp\refresh_copy.log
echo Exit code: %ERRORLEVEL%
```

Exit codes 0 to 7 are success; 8 or higher means files failed to copy. The toolkit's [Copy-BackupFiles.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/robocopy/Copy-BackupFiles.ps1) wraps this with the exit-code check.

Then check the file on TSTSQL01 and note the logical file names for Step 8:

```sql
-- Run on TSTSQL01
RESTORE VERIFYONLY FROM DISK = N'X:\SQLRefresh\AppDB_FULL_20261024.bak' WITH CHECKSUM;
RESTORE HEADERONLY   FROM DISK = N'X:\SQLRefresh\AppDB_FULL_20261024.bak';   -- server, database, backup date
RESTORE FILELISTONLY FROM DISK = N'X:\SQLRefresh\AppDB_FULL_20261024.bak';   -- LogicalName for each file
```

Leave out `WITH CHECKSUM` if the backup was taken without checksums (has_backup_checksums = 0 in Step 4).

<!-- Screenshot: save as docs/images/database-refresh/step-06.png and uncomment the next line -->
<!-- ![Step 6: Copy the backup to the target and check it](../images/database-refresh/step-06.png) -->

> [!TIP]
> **Expected result:** Robocopy exit code below 8, *The backup set on file 1 is valid*, HEADERONLY shows PRDSQL01 / AppDB at the expected date, and FILELISTONLY lists the logical names.

## Step 7: Safety backup of the target and pause its jobs

Take a backup of the target database **as it is now**. It's your rollback if the refresh has to be undone (Step 15).

```sql
-- Run on TSTSQL01
BACKUP DATABASE AppDB
TO DISK = N'X:\SQLRefresh\AppDB_TST_before_refresh.bak'
WITH COPY_ONLY, CHECKSUM, COMPRESSION, INIT, STATS = 10;

-- Agent jobs on TSTSQL01 that use AppDB. Disable the enabled ones until Step 12.
SELECT DISTINCT j.name AS job_name, j.enabled
FROM msdb.dbo.sysjobs AS j
JOIN msdb.dbo.sysjobsteps AS s ON s.job_id = j.job_id
WHERE s.database_name = N'AppDB' OR s.command LIKE N'%AppDB%';

-- For each enabled job listed:
-- EXEC msdb.dbo.sp_update_job @job_name = N'<job name>', @enabled = 0;
```

Write down which jobs you disabled.

<!-- Screenshot: save as docs/images/database-refresh/step-07.png and uncomment the next line -->
<!-- ![Step 7: Safety backup of the target and pause its jobs](../images/database-refresh/step-07.png) -->

> [!TIP]
> **Expected result:** The safety backup finishes, and the jobs that use AppDB are disabled and listed in the ticket.

## Step 8: Restore over the target

Use the logical names from Step 6 and the target paths from Step 3. Run the whole block together in one window on **TSTSQL01**, so no other connection can take the single-user slot between the two statements. `RESTRICTED_USER` keeps the application out until you open it in Step 14.

```sql
-- Run on TSTSQL01
USE master;

-- Disconnect everyone (skip if AppDB doesn't exist on TSTSQL01 yet)
ALTER DATABASE AppDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;

RESTORE DATABASE AppDB
FROM DISK = N'X:\SQLRefresh\AppDB_FULL_20261024.bak'
WITH MOVE N'AppDB'     TO N'D:\SQLData\AppDB.mdf',      -- one MOVE per file in FILELISTONLY
     MOVE N'AppDB_log' TO N'L:\SQLLogs\AppDB_log.ldf',
     REPLACE, RESTRICTED_USER, RECOVERY, STATS = 5;
```

Watch progress from another window with [backup-restore-progress.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/backup-restore/backup-restore-progress.sql). For a point-in-time restore, use `NORECOVERY` on the full and differential, then restore the logs with `STOPAT = '2026-10-24T06:00:00'` and `RECOVERY` on the last one.

If the restore fails and leaves AppDB in SINGLE_USER, set it back with `ALTER DATABASE AppDB SET MULTI_USER;`.

<!-- Screenshot: save as docs/images/database-refresh/step-08.png and uncomment the next line -->
<!-- ![Step 8: Restore over the target](../images/database-refresh/step-08.png) -->

> [!TIP]
> **Expected result:** *RESTORE DATABASE successfully processed*, and AppDB shows as Restricted User in SSMS.

## Step 9: Put the target's database settings back

The restored database carries production's settings. Set it back to the target's standard, using what you saved in Step 3. Run on **TSTSQL01**:

```sql
-- Run on TSTSQL01. Use the values you saved in Step 3.
ALTER AUTHORIZATION ON DATABASE::AppDB TO [sa];                -- your standard database owner
ALTER DATABASE AppDB SET RECOVERY SIMPLE;                       -- your non-production standard
ALTER DATABASE AppDB SET TRUSTWORTHY OFF;

-- Check the rest against Step 3
SELECT  name, SUSER_SNAME(owner_sid) AS owner_name, recovery_model_desc, compatibility_level,
        is_trustworthy_on, is_broker_enabled, is_query_store_on,
        is_cdc_enabled, is_published, is_subscribed, user_access_desc
FROM sys.databases
WHERE name = N'AppDB';
```

If production uses **Service Broker** and the application needs it in this environment, give the database a new broker identifier so it can't exchange messages with production services:

```sql
ALTER DATABASE AppDB SET NEW_BROKER WITH ROLLBACK IMMEDIATE;
```

A restore to another server drops replication and change data capture settings by default, so is_cdc_enabled, is_published and is_subscribed should be 0. If the target needs CDC, enable it again here.

<!-- Screenshot: save as docs/images/database-refresh/step-09.png and uncomment the next line -->
<!-- ![Step 9: Put the target's database settings back](../images/database-refresh/step-09.png) -->

> [!TIP]
> **Expected result:** Owner, recovery model, trustworthy and broker match the target's standard or the Step 3 values, and CDC and replication flags are 0 unless the target needs them.

## Step 10: Remove production users and put the target's access back

The database now has production's users. Users whose production login doesn't exist on TSTSQL01 are orphaned. Run in AppDB on **TSTSQL01**:

```sql
USE AppDB;

-- Every user in the restored database and whether it maps to a login on this server
SELECT  dp.name AS user_name, dp.type_desc, sp.name AS login_name,
        CASE WHEN dp.authentication_type_desc IN (N'NONE', N'DATABASE') THEN N'NO LOGIN BY DESIGN'
             WHEN sp.sid IS NULL                                         THEN N'ORPHANED'
             ELSE N'MAPPED' END AS status,
        (SELECT COUNT(*) FROM sys.schemas AS s WHERE s.principal_id = dp.principal_id) AS schemas_owned
FROM sys.database_principals AS dp
LEFT JOIN sys.server_principals AS sp ON sp.sid = dp.sid
WHERE dp.type IN ('S', 'U', 'G') AND dp.principal_id > 4
ORDER BY status, dp.name;
```

Drop the orphaned production users. This lists the statements first; set `@Execute = 1` to run them. Users that own a schema are skipped: change the schema owner first (`ALTER AUTHORIZATION ON SCHEMA::<schema> TO dbo;`), then run again.

```sql
USE AppDB;
DECLARE @Execute bit = 0;
DECLARE @sql nvarchar(max);

SELECT @sql = STRING_AGG(CAST(N'DROP USER ' + QUOTENAME(dp.name) + N';' AS nvarchar(max)), CHAR(10))
FROM sys.database_principals AS dp
LEFT JOIN sys.server_principals AS sp ON sp.sid = dp.sid
WHERE dp.type IN ('S', 'U', 'G') AND dp.principal_id > 4
  AND sp.sid IS NULL
  AND dp.authentication_type_desc NOT IN (N'NONE', N'DATABASE')
  AND NOT EXISTS (SELECT 1 FROM sys.schemas AS s WHERE s.principal_id = dp.principal_id);

PRINT ISNULL(@sql, N'-- No orphaned users to drop');
IF @Execute = 1 AND @sql IS NOT NULL EXEC sys.sp_executesql @sql;
```

Then:

1. Review the **MAPPED** users. A production AD group that also exists as a login on TSTSQL01 keeps its production rights here. Drop any that shouldn't have access in this environment
2. Run the saved `AppDB_TST_permissions.sql` from Step 3. It skips users that already exist and re-creates the target's users, roles and permissions
3. Run [orphaned-users-fix.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/security/orphaned-users-fix.sql) for any SQL users still orphaned because the login exists with a different SID

<!-- Screenshot: save as docs/images/database-refresh/step-10.png and uncomment the next line -->
<!-- ![Step 10: Remove production users and put the target's access back](../images/database-refresh/step-10.png) -->

> [!TIP]
> **Expected result:** Running the first query again shows no ORPHANED users, and the MAPPED users match the target's access from Step 3.

## Step 11: Mask the data and remove links to production

Run the **approved masking script** from the application or data team. Don't write your own on the day: the data owner decides what is sensitive and how it is masked. A typical script overwrites names, email addresses, phone numbers, account numbers and government ID numbers, for example:

```sql
-- Example only. Use the approved masking script for AppDB.
UPDATE dbo.Customer
SET Email       = CONCAT(N'user', CustomerID, N'@masked.invalid'),
    PhoneNumber = N'0000000000';
```

Then look for anything that still points at production:

```sql
USE AppDB;

-- Synonyms that point at another server (four-part names)
SELECT SCHEMA_NAME(schema_id) AS schema_name, name, base_object_name
FROM sys.synonyms
WHERE base_object_name LIKE N'%.%.%.%';

-- Procedures, views, functions and triggers that name the production server
SELECT OBJECT_SCHEMA_NAME(m.object_id) AS schema_name, OBJECT_NAME(m.object_id) AS object_name, o.type_desc
FROM sys.sql_modules AS m
JOIN sys.objects AS o ON o.object_id = m.object_id
WHERE m.definition LIKE N'%PRDSQL01%';
```

Ask the application team to update any configuration tables that hold production URLs, file paths, email recipients or API endpoints. Reload any target-only data you exported in Step 3.

<!-- Screenshot: save as docs/images/database-refresh/step-11.png and uncomment the next line -->
<!-- ![Step 11: Mask the data and remove links to production](../images/database-refresh/step-11.png) -->

> [!TIP]
> **Expected result:** The masking script completed with the expected row counts, and both queries return no rows (or each row has been changed to point at the target environment).

## Step 12: Re-enable the target's jobs

Turn back on the jobs you disabled in Step 7:

```sql
-- Run on TSTSQL01 for each job you disabled in Step 7
EXEC msdb.dbo.sp_update_job @job_name = N'<job name>', @enabled = 1;
```

Production's Agent jobs live in production's msdb and are **not** copied by a database restore, so nothing from production starts running on TSTSQL01.

<!-- Screenshot: save as docs/images/database-refresh/step-12.png and uncomment the next line -->
<!-- ![Step 12: Re-enable the target's jobs](../images/database-refresh/step-12.png) -->

> [!TIP]
> **Expected result:** Every job disabled in Step 7 is enabled again.

## Step 13: Validate the refresh

Run on **TSTSQL01**:

```sql
-- 1. The refresh came from the right backup
SELECT TOP (1) rh.destination_database_name, rh.restore_date,
       bs.server_name AS source_server, bs.database_name AS source_database,
       bs.backup_finish_date, bmf.physical_device_name
FROM msdb.dbo.restorehistory AS rh
JOIN msdb.dbo.backupset AS bs ON bs.backup_set_id = rh.backup_set_id
JOIN msdb.dbo.backupmediafamily AS bmf ON bmf.media_set_id = bs.media_set_id
WHERE rh.destination_database_name = N'AppDB'
ORDER BY rh.restore_date DESC;

-- 2. The application login can connect and read (impersonation test)
USE AppDB;
EXECUTE AS LOGIN = N'AppLogin';
SELECT SUSER_SNAME() AS login_name, USER_NAME() AS database_user, COUNT(*) AS tables_visible
FROM sys.tables;
REVERT;
```

Then:

- Run [schema-object-counts.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/maintenance/schema-object-counts.sql) in AppDB on **both** PRDSQL01 and TSTSQL01 and compare. Object counts should match; row counts should match as of the backup time
- Spot-check a few key tables with the requester (for example the latest transaction date matches the backup date)

<!-- Screenshot: save as docs/images/database-refresh/step-13.png and uncomment the next line -->
<!-- ![Step 13: Validate the refresh](../images/database-refresh/step-13.png) -->

> [!TIP]
> **Expected result:** Restore history shows PRDSQL01 / AppDB at the expected backup date, AppLogin connects as its target database user and can see the tables, and object counts match production.

## Step 14: Open the database, back it up and clean up

```sql
-- Run on TSTSQL01
ALTER DATABASE AppDB SET MULTI_USER;

-- New baseline backup on the target (the recovery model changed and the old chain no longer applies)
BACKUP DATABASE AppDB
TO DISK = N'X:\SQLBackups\AppDB_FULL_after_refresh.bak'
WITH CHECKSUM, COMPRESSION, INIT, STATS = 10;
```

Delete every copy of the unmasked production backup:

```powershell
Remove-Item 'X:\SQLRefresh\AppDB_FULL_20261024.bak'
Remove-Item '\\backupshare\SQLRefresh\AppDB_FULL_20261024.bak'
```

Keep `AppDB_TST_before_refresh.bak` until the requester signs off, then delete it too. Update the ticket and tell the requester the database is ready.

<!-- Screenshot: save as docs/images/database-refresh/step-14.png and uncomment the next line -->
<!-- ![Step 14: Open the database, back it up and clean up](../images/database-refresh/step-14.png) -->

> [!TIP]
> **Expected result:** AppDB is MULTI_USER, a new full backup exists on the target, and no copy of the production backup is left on the share or on TSTSQL01.

## Step 15: Roll back if needed

If the refresh has to be undone, restore the safety backup from Step 7. The target's users and settings come back with it.

```sql
-- Run on TSTSQL01
USE master;
ALTER DATABASE AppDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;

RESTORE DATABASE AppDB
FROM DISK = N'X:\SQLRefresh\AppDB_TST_before_refresh.bak'
WITH REPLACE, RECOVERY, STATS = 5;

ALTER DATABASE AppDB SET MULTI_USER;
```

<!-- Screenshot: save as docs/images/database-refresh/step-15.png and uncomment the next line -->
<!-- ![Step 15: Roll back if needed](../images/database-refresh/step-15.png) -->

> [!TIP]
> **Expected result:** AppDB is back as it was before the refresh, and the application team confirms it works.

## Validation sign-off

- [ ] Request, data owner approval and masking approval attached to the ticket
- [ ] Target settings, file paths and permissions script saved before the refresh
- [ ] Source backup taken COPY_ONLY (or an existing backup used and named in the ticket)
- [ ] TDE certificate present on the target, with security approval if it was copied
- [ ] Safety backup of the target taken before the restore
- [ ] Restore history shows the right source server, database and backup date
- [ ] Owner, recovery model and trustworthy set to the target standard; Service Broker handled
- [ ] No orphaned users; production-only access removed; target permissions re-applied
- [ ] Approved masking script run; no synonyms or code still pointing at production
- [ ] AppLogin impersonation test passed; object counts match production
- [ ] Jobs re-enabled; database MULTI_USER; new full backup taken
- [ ] Every copy of the production backup deleted; requester sign-off received

## Sources

- **Microsoft Learn:** [Restore a database to a new location (SQL Server)](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/restore-a-database-to-a-new-location-sql-server)
- **Microsoft Learn:** [RESTORE statements (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/restore-statements-transact-sql)
- **Microsoft Learn:** [Copy-only backups](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/copy-only-backups-sql-server)
- **Microsoft Learn:** [Move a TDE-protected database to another SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/security/encryption/move-a-tde-protected-database-to-another-sql-server)
- **Microsoft Learn:** [Troubleshoot orphaned users (SQL Server)](https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/troubleshoot-orphaned-users-sql-server)
- **MSSQLTips:** [Automating a SQL Server database refresh](https://www.mssqltips.com/sqlservertip/5465/automating-a-sql-server-database-refresh/)
- **SQLShack:** [Automatically refresh a SQL Server database](https://www.sqlshack.com/automatically-refresh-sql-server-database/)
- **Toolkit:** [MySqlScripts](https://github.com/Aderemil28/MySqlScripts)

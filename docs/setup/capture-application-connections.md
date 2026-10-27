# Capture which applications connect to each database

**Status:** Draft · **Applies to:** SQL Server 2016 to 2022 · **Steps:** 7

Record, over days or weeks, which logins, hosts, programs and IP addresses connect to each database on an instance. Use it for **application ownership discovery and dependency mapping**: before a decommission, a migration, a password or service account change, or when nobody knows who still uses a database.

A SQL Agent job takes a snapshot of current connections on a schedule and stores a summary in a table. It only reads the session DMVs, so it has no noticeable impact on the workload.

> [!NOTE]
> **Before you start**
>
> - sysadmin on the instance (or VIEW SERVER STATE plus rights to create a job and a table)
> - A DBA utility database to hold the table. This runbook uses **DBAUtil**; use your own
> - The login your DBA team connects with, so it can be left out of the results (**DBA_LOGIN** below)
> - On an **availability group**, connections go to whichever replica the application points at: set this up on **every replica**
> - The table records logins, hosts and IP addresses. Keep access to it limited to the DBA team

## Step 1: Create the table

One row per distinct login, host, program, database and client address in each capture, with how many sessions it had. That's far fewer rows than storing every session, and it's all you need to answer "who uses this database?".

```sql
USE DBAUtil;
GO
CREATE TABLE dbo.ApplicationConnections
(
    CaptureTime      datetime2(0)   NOT NULL CONSTRAINT DF_ApplicationConnections_CaptureTime DEFAULT SYSDATETIME(),
    ServerName       sysname        NOT NULL,
    DatabaseName     sysname        NULL,
    LoginName        sysname        NOT NULL,
    HostName         sysname        NULL,
    ProgramName      nvarchar(256)  NULL,
    ClientAddress    varchar(48)    NULL,
    AuthScheme       nvarchar(40)   NULL,
    SessionCount     int            NOT NULL,
    LastRequestEnd   datetime2(0)   NULL
);
CREATE CLUSTERED INDEX CIX_ApplicationConnections_CaptureTime ON dbo.ApplicationConnections (CaptureTime);
CREATE INDEX IX_ApplicationConnections_Database ON dbo.ApplicationConnections (DatabaseName, LoginName);
```

<!-- Screenshot: save as docs/images/capture-application-connections/step-01.png and uncomment the next line -->
<!-- ![Step 1: Create the table](../images/capture-application-connections/step-01.png) -->

> [!TIP]
> **Expected result:** `dbo.ApplicationConnections` exists in DBAUtil.

## Step 2: Test the capture query

Run the query the job will run, to see what it records. Replace `DBA_LOGIN` with your team's login (add more with `NOT IN`).

```sql
SELECT  SYSDATETIME()                                           AS CaptureTime,
        @@SERVERNAME                                            AS ServerName,
        DB_NAME(COALESCE(r.database_id, s.database_id))         AS DatabaseName,
        s.login_name                                            AS LoginName,
        s.host_name                                             AS HostName,
        s.program_name                                          AS ProgramName,
        c.client_net_address                                    AS ClientAddress,
        c.auth_scheme                                           AS AuthScheme,
        COUNT(*)                                                AS SessionCount,
        MAX(s.last_request_end_time)                            AS LastRequestEnd
FROM sys.dm_exec_sessions AS s
LEFT JOIN sys.dm_exec_requests    AS r ON r.session_id = s.session_id
LEFT JOIN sys.dm_exec_connections AS c ON c.session_id = s.session_id
WHERE s.is_user_process = 1
  AND s.login_name NOT IN (N'DBA_LOGIN')
GROUP BY DB_NAME(COALESCE(r.database_id, s.database_id)), s.login_name, s.host_name,
         s.program_name, c.client_net_address, c.auth_scheme;
```

`auth_scheme` shows **NTLM**, **KERBEROS** or **SQL**, which helps when planning a service account or Kerberos change.

<!-- Screenshot: save as docs/images/capture-application-connections/step-02.png and uncomment the next line -->
<!-- ![Step 2: Test the capture query](../images/capture-application-connections/step-02.png) -->

> [!TIP]
> **Expected result:** One row per application connection pattern, without your own DBA sessions.

## Step 3: Create the Agent job

The job has two steps: capture, then delete rows older than the retention period so the table can't grow forever. It runs **every hour** by default; use every 15 minutes if short-lived connections are missed.

```sql
USE msdb;
SET NOCOUNT ON;

DECLARE @JobName      sysname = N'DBA - Capture application connections';
DECLARE @DbaLogin     sysname = N'DBA_LOGIN';   -- left out of the results
DECLARE @EveryMinutes int     = 60;
DECLARE @KeepDays     int     = 90;
DECLARE @jobId uniqueidentifier, @rc int, @capture nvarchar(max), @purge nvarchar(max);

SET @capture = N'INSERT INTO dbo.ApplicationConnections
    (CaptureTime, ServerName, DatabaseName, LoginName, HostName, ProgramName, ClientAddress, AuthScheme, SessionCount, LastRequestEnd)
SELECT SYSDATETIME(), @@SERVERNAME, DB_NAME(COALESCE(r.database_id, s.database_id)), s.login_name, s.host_name,
       s.program_name, c.client_net_address, c.auth_scheme, COUNT(*), MAX(s.last_request_end_time)
FROM sys.dm_exec_sessions AS s
LEFT JOIN sys.dm_exec_requests    AS r ON r.session_id = s.session_id
LEFT JOIN sys.dm_exec_connections AS c ON c.session_id = s.session_id
WHERE s.is_user_process = 1
  AND s.login_name <> N''' + REPLACE(@DbaLogin, N'''', N'''''') + N'''
GROUP BY DB_NAME(COALESCE(r.database_id, s.database_id)), s.login_name, s.host_name,
         s.program_name, c.client_net_address, c.auth_scheme;';

SET @purge = N'DELETE FROM dbo.ApplicationConnections
WHERE CaptureTime < DATEADD(DAY, -' + CAST(@KeepDays AS nvarchar(10)) + N', SYSDATETIME());';

BEGIN TRANSACTION;

EXEC @rc = msdb.dbo.sp_add_job
     @job_name = @JobName, @enabled = 1, @owner_login_name = N'sa',
     @description = N'Captures read-only session metadata (login, host, program, database, client address) to identify which applications use each database. Used for application ownership discovery and dependency mapping. No workload impact.',
     @job_id = @jobId OUTPUT;
IF @@ERROR <> 0 OR @rc <> 0 GOTO QuitWithRollback;

EXEC @rc = msdb.dbo.sp_add_jobstep
     @job_id = @jobId, @step_id = 1, @step_name = N'Capture connections',
     @subsystem = N'TSQL', @database_name = N'DBAUtil', @command = @capture,
     @on_success_action = 3, @on_fail_action = 3;     -- go on to the purge either way
IF @@ERROR <> 0 OR @rc <> 0 GOTO QuitWithRollback;

EXEC @rc = msdb.dbo.sp_add_jobstep
     @job_id = @jobId, @step_id = 2, @step_name = N'Purge old rows',
     @subsystem = N'TSQL', @database_name = N'DBAUtil', @command = @purge,
     @on_success_action = 1, @on_fail_action = 2;
IF @@ERROR <> 0 OR @rc <> 0 GOTO QuitWithRollback;

EXEC @rc = msdb.dbo.sp_add_jobschedule
     @job_id = @jobId, @name = N'Every N minutes', @enabled = 1,
     @freq_type = 4, @freq_interval = 1,                       -- daily
     @freq_subday_type = 4, @freq_subday_interval = @EveryMinutes;  -- every N minutes
IF @@ERROR <> 0 OR @rc <> 0 GOTO QuitWithRollback;

EXEC @rc = msdb.dbo.sp_add_jobserver @job_id = @jobId, @server_name = N'(local)';
IF @@ERROR <> 0 OR @rc <> 0 GOTO QuitWithRollback;

COMMIT TRANSACTION;
PRINT 'Created ' + @JobName;
GOTO EndSave;
QuitWithRollback:
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
    PRINT 'Failed; nothing was created.';
EndSave:
```

Two things to check if you script a job like this out of SSMS instead:

- **The job and its schedule are enabled separately.** A job scripted with `@enabled=0` never runs, even with an enabled schedule
- **Remove the `@schedule_uid` line** before running the script on another server. SSMS includes the original schedule's ID, which belongs to the server it came from

<!-- Screenshot: save as docs/images/capture-application-connections/step-03.png and uncomment the next line -->
<!-- ![Step 3: Create the Agent job](../images/capture-application-connections/step-03.png) -->

> [!TIP]
> **Expected result:** `Created DBA - Capture application connections`, and the job shows under SQL Server Agent > Jobs, enabled.

## Step 4: Run it once and check the rows

```sql
EXEC msdb.dbo.sp_start_job @job_name = N'DBA - Capture application connections';
WAITFOR DELAY '00:00:10';

SELECT TOP (20) * FROM DBAUtil.dbo.ApplicationConnections ORDER BY CaptureTime DESC;
```

Check the job history if no rows appear (`tsql/agent-jobs/failed-jobs.sql` in MySqlScripts).

<!-- Screenshot: save as docs/images/capture-application-connections/step-04.png and uncomment the next line -->
<!-- ![Step 4: Run it once and check the rows](../images/capture-application-connections/step-04.png) -->

> [!TIP]
> **Expected result:** Rows with the current time, one per connection pattern.

## Step 5: Let it run long enough

Leave it running long enough to catch everything that connects: at least a **full business cycle**, including a month end. Batch jobs, monthly reports and DR tests often connect only a few times a year, so the longer it runs before a decommission, the safer the decision.

## Step 6: Report who uses each database

```sql
USE DBAUtil;

-- Every application pattern per database, with when it was first and last seen
SELECT  DatabaseName,
        LoginName,
        HostName,
        ProgramName,
        ClientAddress,
        AuthScheme,
        MIN(CaptureTime)     AS FirstSeen,
        MAX(CaptureTime)     AS LastSeen,
        COUNT(*)             AS TimesSeen,
        MAX(SessionCount)    AS PeakSessions
FROM dbo.ApplicationConnections
GROUP BY DatabaseName, LoginName, HostName, ProgramName, ClientAddress, AuthScheme
ORDER BY DatabaseName, LastSeen DESC;

-- Databases nobody has connected to since the capture started
SELECT d.name AS DatabaseName
FROM sys.databases AS d
WHERE d.database_id > 4
  AND d.name NOT IN (SELECT DatabaseName FROM dbo.ApplicationConnections WHERE DatabaseName IS NOT NULL)
ORDER BY d.name;
```

A database appears here only if a session had it as its current database. An application that connects to one database and queries another with three-part names shows under the first one, so use the results as a strong lead and confirm with the application owner.

<!-- Screenshot: save as docs/images/capture-application-connections/step-06.png and uncomment the next line -->
<!-- ![Step 6: Report who uses each database](../images/capture-application-connections/step-06.png) -->

> [!TIP]
> **Expected result:** A list you can send to application owners: which login, from which host and program, last connected when.

## Step 7: Stop and clean up when done

```sql
EXEC msdb.dbo.sp_update_job @job_name = N'DBA - Capture application connections', @enabled = 0;   -- pause

-- When the results are no longer needed:
-- EXEC msdb.dbo.sp_delete_job @job_name = N'DBA - Capture application connections';
-- DROP TABLE DBAUtil.dbo.ApplicationConnections;
```

> [!TIP]
> **Expected result:** The job is disabled (or removed) and, once the results are saved elsewhere, the table is dropped.

## Validation sign-off

- [ ] Table created in the DBA utility database, access limited to DBAs
- [ ] Capture query tested; DBA logins left out
- [ ] Job created, **enabled**, with a schedule, and the purge step in place
- [ ] First run produced rows
- [ ] Set up on every AG replica, if the database is in an availability group
- [ ] Ran for a full business cycle before decisions were made
- [ ] Report sent to the application owners; job disabled or removed afterwards

## Sources

- **Microsoft Learn:** [sys.dm_exec_sessions](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-sessions-transact-sql)
- **Microsoft Learn:** [sys.dm_exec_connections](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-connections-transact-sql)
- **Microsoft Learn:** [sp_add_jobschedule](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-add-jobschedule-transact-sql)
- **Toolkit:** [sessions-by-database-host-or-program.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/monitoring/sessions-by-database-host-or-program.sql) for a one-off check instead of a capture

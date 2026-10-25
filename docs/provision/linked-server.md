# Create a linked server (SQL Server to SQL Server)

**Status:** Draft · **Applies to:** SQL Server 2016 to 2022 · **Steps:** 12

Set up a linked server so queries on one instance can read from (or run procedures on) another. In this runbook **SQL01** is the server where you create the link and **SQL02** is the remote server it points to. Replace both with your own server names.

> [!NOTE]
> **Before you start**
>
> - sysadmin (or ALTER ANY LINKED SERVER plus ALTER ANY LOGIN) on SQL01
> - Rights to create a login on SQL02
> - A clear answer to: which database on SQL02, read-only or read-write, and does it need to run stored procedures there?
> - TCP 1433 (or the instance port) open from SQL01 to SQL02
> - A change ticket if this is not a lab. Linked servers are a common audit finding, so record who asked for it and why

## Step 1: Choose the security model

This decides who the remote server thinks is connecting. Pick one before you build anything:

- **Mapped remote login (recommended)**: every query over the link runs on SQL02 as one dedicated, least-privilege login. Simple and predictable
- **Be made using the login's current security context**: each user connects as themselves. Needs Kerberos (SPNs and delegation) to work across servers, otherwise you get *Login failed for user 'NT AUTHORITY\ANONYMOUS LOGON'*
- Never map to **sa** or a sysadmin login, and never use **Be made without using a security context** for anything real

The steps below use the mapped remote login.

<!-- Screenshot: save as docs/images/linked-server/step-01.png and uncomment the next line -->
<!-- ![Step 1: Choose the security model](../images/linked-server/step-01.png) -->

## Step 2: On SQL02: create the remote login

Create a dedicated login and give it only the database access the link needs.

```sql
-- Run on SQL02
CREATE LOGIN lnk_from_sql01 WITH PASSWORD = N'<StrongPassword>', CHECK_POLICY = ON;

USE YourDatabase;
CREATE USER lnk_from_sql01 FOR LOGIN lnk_from_sql01;
ALTER ROLE db_datareader ADD MEMBER lnk_from_sql01;      -- read-only
-- GRANT EXECUTE ON SCHEMA::dbo TO lnk_from_sql01;      -- only if it must run procedures
```

<!-- Screenshot: save as docs/images/linked-server/step-02.png and uncomment the next line -->
<!-- ![Step 2: On SQL02: create the remote login](../images/linked-server/step-02.png) -->

> [!TIP]
> **Expected result:** The login exists on SQL02 and is a member of db_datareader in the target database only.

## Step 3: On SQL01: check you can reach SQL02

Rule out network problems before blaming the linked server. Run in PowerShell on SQL01:

```powershell
Test-NetConnection -ComputerName SQL02 -Port 1433

# Is the Microsoft OLE DB Driver for SQL Server installed? (SQL Server 2022 includes it)
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*' |
    Where-Object DisplayName -like '*OLE DB Driver*SQL Server*' |
    Select-Object DisplayName, DisplayVersion
```

<!-- Screenshot: save as docs/images/linked-server/step-03.png and uncomment the next line -->
<!-- ![Step 3: On SQL01: check you can reach SQL02](../images/linked-server/step-03.png) -->

> [!TIP]
> **Expected result:** TcpTestSucceeded : True, and an OLE DB Driver for SQL Server entry is listed.

## Step 4: Create the linked server in SSMS (option A)

1. Connect to **SQL01** in SSMS
2. Expand **Server Objects**, right-click **Linked Servers** > **New Linked Server**
3. **General** page: set **Linked server** to `SQL02` and **Server type** to **SQL Server**
4. **Security** page: choose **Be made using this security context** and enter `lnk_from_sql01` and its password
5. **Server Options** page: leave **Data Access** = True; set **RPC Out** = True only if you will run stored procedures on SQL02; set **Enable Promotion of Distributed Transactions** = False unless you need distributed transactions
6. Click **OK**

Add a screenshot of each page here as you go.

<!-- Screenshot: save as docs/images/linked-server/step-04.png and uncomment the next line -->
<!-- ![Step 4: Create the linked server in SSMS (option A)](../images/linked-server/step-04.png) -->

> [!TIP]
> **Expected result:** SQL02 appears under Server Objects > Linked Servers with no error on save.

## Step 5: Or create it with T-SQL (option B)

The script version is easier to repeat on another server and to attach to a change record. Run on SQL01:

```sql
-- Run on SQL01
EXEC master.dbo.sp_addlinkedserver
     @server     = N'SQL02',          -- name you will use in queries
     @srvproduct = N'',
     @provider   = N'MSOLEDBSQL',     -- Microsoft OLE DB Driver for SQL Server
     @datasrc    = N'SQL02';          -- network name; add ,port or \instance if needed

EXEC master.dbo.sp_addlinkedsrvlogin
     @rmtsrvname  = N'SQL02',
     @useself     = N'False',
     @locallogin  = NULL,             -- NULL = applies to every local login
     @rmtuser     = N'lnk_from_sql01',
     @rmtpassword = N'<StrongPassword>';

EXEC master.dbo.sp_serveroption N'SQL02', 'data access', 'true';
EXEC master.dbo.sp_serveroption N'SQL02', 'rpc out', 'false';   -- 'true' only if running procedures remotely
EXEC master.dbo.sp_serveroption N'SQL02', 'remote proc transaction promotion', 'false';
```

<!-- Screenshot: save as docs/images/linked-server/step-05.png and uncomment the next line -->
<!-- ![Step 5: Or create it with T-SQL (option B)](../images/linked-server/step-05.png) -->

> [!TIP]
> **Expected result:** All statements complete with no errors.

## Step 6: Limit who can use the link (optional)

With `@locallogin = NULL`, every login on SQL01 can use the link. To allow only one application login, map that login and block everyone else:

```sql
-- Run on SQL01: allow only AppLogin to use the link
EXEC master.dbo.sp_droplinkedsrvlogin @rmtsrvname = N'SQL02', @locallogin = NULL;

EXEC master.dbo.sp_addlinkedsrvlogin
     @rmtsrvname  = N'SQL02',
     @useself     = N'False',
     @locallogin  = N'AppLogin',
     @rmtuser     = N'lnk_from_sql01',
     @rmtpassword = N'<StrongPassword>';
```

<!-- Screenshot: save as docs/images/linked-server/step-06.png and uncomment the next line -->
<!-- ![Step 6: Limit who can use the link (optional)](../images/linked-server/step-06.png) -->

> [!TIP]
> **Expected result:** Other logins now get a login-failed error when they query SQL02 through the link.

## Step 7: Test the link

Test from SQL01, first the connection, then a real query.

```sql
-- Run on SQL01
EXEC master.dbo.sp_testlinkedserver N'SQL02';

-- Four-part name: server.database.schema.object
SELECT TOP (10) * FROM SQL02.YourDatabase.dbo.YourTable;

-- Pass-through query (runs entirely on SQL02, usually faster for filters)
SELECT * FROM OPENQUERY(SQL02, 'SELECT @@SERVERNAME AS remote_server, SUSER_SNAME() AS remote_login');
```

<!-- Screenshot: save as docs/images/linked-server/step-07.png and uncomment the next line -->
<!-- ![Step 7: Test the link](../images/linked-server/step-07.png) -->

> [!TIP]
> **Expected result:** sp_testlinkedserver completes with no error, rows come back, and remote_login shows lnk_from_sql01.

## Step 8: If you get a certificate error

The OLE DB Driver version 19 and later encrypts by default and checks the remote server's certificate. If SQL02 uses a self-signed certificate you may see *The certificate chain was issued by an authority that is not trusted*.

- **Best fix**: install a certificate on SQL02 that SQL01 trusts
- **Lab or short-term**: recreate the linked server with a provider string that trusts the certificate. This still encrypts but does not verify who SQL02 is, so get it approved before using it in production

```sql
-- Recreate with an explicit provider string (lab / approved exceptions only)
EXEC master.dbo.sp_dropserver N'SQL02', 'droplogins';

EXEC master.dbo.sp_addlinkedserver
     @server     = N'SQL02',
     @srvproduct = N'',
     @provider   = N'MSOLEDBSQL',
     @datasrc    = N'SQL02',
     @provstr    = N'Encrypt=Yes;TrustServerCertificate=Yes';
-- then repeat sp_addlinkedsrvlogin and sp_serveroption from step 5
```

<!-- Screenshot: save as docs/images/linked-server/step-08.png and uncomment the next line -->
<!-- ![Step 8: If you get a certificate error](../images/linked-server/step-08.png) -->

> [!TIP]
> **Expected result:** sp_testlinkedserver succeeds after the certificate is trusted or the provider string is applied.

## Step 9: Run the validation script

This checks that the linked server is built the way it was asked for, not just that it works. Run it on **SQL01** as a DBA. First find and replace `SQL02`, `YourDatabase` and `lnk_from_sql01` with your own names. Each check is its own batch, so one failure does not stop the rest, and each returns a `result` column: **PASS**, **FAIL** or **REVIEW**.

```sql
/* Linked server validation. Run on SQL01.
   Find and replace: SQL02, YourDatabase, lnk_from_sql01 */
USE master;
SET NOCOUNT ON;
GO

-- Check 1: configuration (provider, data access, distributed transaction promotion)
SELECT  N'1. Configuration'                                AS check_name,
        s.name                                             AS linked_server,
        s.provider, s.data_source, s.provider_string,
        s.is_data_access_enabled                           AS data_access,
        s.is_rpc_out_enabled                               AS rpc_out,
        s.is_remote_proc_transaction_promotion_enabled     AS dtc_promotion,
        CASE WHEN s.provider LIKE N'MSOLEDBSQL%'
              AND s.is_data_access_enabled = 1
              AND s.is_remote_proc_transaction_promotion_enabled = 0
             THEN N'PASS' ELSE N'REVIEW' END               AS result
FROM sys.servers AS s
WHERE s.name = N'SQL02' AND s.is_linked = 1;
GO

-- Check 2: login mappings (who on SQL01 can use the link, and as whom)
SELECT  N'2. Login mapping'                                AS check_name,
        CASE WHEN l.local_principal_id = 0 THEN N'<all logins>'
             ELSE SUSER_NAME(l.local_principal_id) END     AS local_login,
        l.uses_self_credential,
        l.remote_name                                      AS remote_login,
        CASE WHEN l.remote_name = N'sa'                                THEN N'FAIL'   -- never map to sa
             WHEN l.uses_self_credential = 0 AND l.remote_name IS NULL THEN N'FAIL'   -- no security context
             WHEN l.local_principal_id = 0                             THEN N'REVIEW' -- every login can use it
             WHEN l.remote_name = N'lnk_from_sql01'                    THEN N'PASS'
             ELSE N'REVIEW' END                            AS result
FROM sys.servers AS s
JOIN sys.linked_logins AS l ON l.server_id = s.server_id
WHERE s.name = N'SQL02';
GO

-- Check 3: connectivity
BEGIN TRY
    EXEC master.dbo.sp_testlinkedserver N'SQL02';
    SELECT N'3. Connectivity' AS check_name, N'PASS' AS result, NULL AS detail;
END TRY
BEGIN CATCH
    SELECT N'3. Connectivity' AS check_name, N'FAIL' AS result,
           CONCAT(ERROR_NUMBER(), N': ', ERROR_MESSAGE()) AS detail;
END CATCH;
GO

-- Check 4: who SQL02 thinks is connecting, and whether the link is encrypted
SELECT  N'4. Remote identity'                              AS check_name,
        r.remote_server, r.remote_login, r.is_sysadmin, r.encrypt_option, r.auth_scheme,
        CASE WHEN r.remote_login <> N'lnk_from_sql01' OR r.is_sysadmin = 1 THEN N'FAIL'
             WHEN r.encrypt_option <> N'TRUE'                              THEN N'REVIEW'
             ELSE N'PASS' END                              AS result
FROM OPENQUERY(SQL02, 'SELECT @@SERVERNAME AS remote_server,
                              SUSER_SNAME() AS remote_login,
                              IS_SRVROLEMEMBER(''sysadmin'') AS is_sysadmin,
                              CAST(CONNECTIONPROPERTY(''encrypt_option'') AS nvarchar(10)) AS encrypt_option,
                              CAST(CONNECTIONPROPERTY(''auth_scheme'') AS nvarchar(40)) AS auth_scheme') AS r;
GO

-- Check 5: what the remote login can do in the target database
-- Read-only links should show db_datareader and CONNECT only.
SELECT  N'5. Remote rights'                                AS check_name,
        p.kind, p.name, p.state,
        CASE WHEN p.state <> N'DENY'
              AND p.name IN (N'db_owner', N'db_datawriter', N'db_ddladmin', N'db_securityadmin',
                             N'db_accessadmin', N'INSERT', N'UPDATE', N'DELETE', N'ALTER', N'CONTROL')
             THEN N'REVIEW' ELSE N'PASS' END               AS result
FROM OPENQUERY(SQL02, 'SELECT N''ROLE'' AS kind, r.name COLLATE DATABASE_DEFAULT AS name, N''MEMBER'' AS state
                       FROM YourDatabase.sys.database_role_members AS rm
                       JOIN YourDatabase.sys.database_principals AS r ON r.principal_id = rm.role_principal_id
                       JOIN YourDatabase.sys.database_principals AS m ON m.principal_id = rm.member_principal_id
                       WHERE m.sid = SUSER_SID()
                       UNION ALL
                       SELECT N''PERMISSION'', dp.permission_name COLLATE DATABASE_DEFAULT,
                              dp.state_desc COLLATE DATABASE_DEFAULT
                       FROM YourDatabase.sys.database_permissions AS dp
                       JOIN YourDatabase.sys.database_principals AS m ON m.principal_id = dp.grantee_principal_id
                       WHERE m.sid = SUSER_SID()') AS p;
GO

-- Check 6: RPC Out really is off (or on, if remote procedures were requested)
BEGIN TRY
    EXEC sys.sp_executesql N'EXEC (''SELECT 1 AS rpc_test'') AT SQL02;';
    SELECT N'6. RPC Out' AS check_name, N'ENABLED' AS state,
           N'PASS only if remote procedures were requested' AS result;
END TRY
BEGIN CATCH
    SELECT N'6. RPC Out' AS check_name, N'DISABLED' AS state,
           CASE WHEN ERROR_NUMBER() = 7411 THEN N'PASS if read-only was requested'
                ELSE CONCAT(N'REVIEW ', ERROR_NUMBER(), N': ', ERROR_MESSAGE()) END AS result;
END CATCH;
GO
```

If Check 3 fails, stop and fix connectivity first: Checks 4 and 5 will fail for the same reason.

<!-- Screenshot: save as docs/images/linked-server/step-09.png and uncomment the next line -->
<!-- ![Step 9: Run the validation script](../images/linked-server/step-09.png) -->

> [!TIP]
> **Expected result:** Checks 1, 3 and 4 show PASS. Check 2 shows PASS for each mapped login (REVIEW only if you deliberately left the link open to all logins). Check 5 lists db_datareader and CONNECT with PASS. Check 6 matches what was requested.

## Step 10: Impersonation testing before handoff

Steps 7 and 9 run as you, a sysadmin, so they prove the link works but not that it works **for the application**, or that it stays closed to everyone else. Impersonation runs the same query as other logins without needing their passwords.

The script tests two logins:

- **AppLogin** (the login the application uses): should be **allowed** and land on SQL02 as `lnk_from_sql01`
- **lnk_test_denied** (a throwaway login with no mapping, created and dropped by the script): should be **refused** with error 7416

Run on **SQL01** as sysadmin, in master. Find and replace `SQL02`, `AppLogin` and `lnk_from_sql01`.

```sql
/* Linked server impersonation test. Run on SQL01 as sysadmin.
   Find and replace: SQL02, AppLogin, lnk_from_sql01 */
USE master;
SET NOCOUNT ON;

-- Throwaway login with no linked server mapping. Dropped at the end.
IF SUSER_ID(N'lnk_test_denied') IS NULL
    CREATE LOGIN lnk_test_denied WITH PASSWORD = N'<RandomStrongPassword>', CHECK_POLICY = ON;

-- If a mapping for <all logins> still exists, every login is allowed, so expect ALLOW for the test login too
DECLARE @HasDefaultMapping bit =
    CASE WHEN EXISTS (SELECT 1
                      FROM sys.linked_logins AS l
                      JOIN sys.servers AS s ON s.server_id = l.server_id
                      WHERE s.name = N'SQL02' AND l.local_principal_id = 0)
         THEN 1 ELSE 0 END;

DECLARE @Cases TABLE (login_name sysname, expected nvarchar(5));
INSERT INTO @Cases (login_name, expected) VALUES
    (N'AppLogin',        N'ALLOW'),
    (N'lnk_test_denied', CASE WHEN @HasDefaultMapping = 1 THEN N'ALLOW' ELSE N'DENY' END);

DECLARE @Results TABLE (login_name sysname, expected nvarchar(5), actual nvarchar(5),
                        remote_login sysname NULL, detail nvarchar(4000) NULL);
DECLARE @login sysname, @expected nvarchar(5), @remote sysname;

DECLARE c CURSOR LOCAL FAST_FORWARD FOR SELECT login_name, expected FROM @Cases;
OPEN c;
FETCH NEXT FROM c INTO @login, @expected;
WHILE @@FETCH_STATUS = 0
BEGIN
    SET @remote = NULL;
    BEGIN TRY
        EXECUTE AS LOGIN = @login;
        EXEC sys.sp_executesql
             N'SELECT @r = remote_login FROM OPENQUERY(SQL02, ''SELECT SUSER_SNAME() AS remote_login'');',
             N'@r sysname OUTPUT', @r = @remote OUTPUT;
        REVERT;
        INSERT INTO @Results VALUES (@login, @expected, N'ALLOW', @remote, NULL);
    END TRY
    BEGIN CATCH
        IF SUSER_SNAME() <> ORIGINAL_LOGIN() REVERT;
        INSERT INTO @Results VALUES (@login, @expected, N'DENY', NULL,
                                     CONCAT(ERROR_NUMBER(), N': ', ERROR_MESSAGE()));
    END CATCH;
    FETCH NEXT FROM c INTO @login, @expected;
END;
CLOSE c;
DEALLOCATE c;

SELECT  login_name, expected, actual, remote_login, detail,
        CASE WHEN expected = N'ALLOW' AND actual = N'ALLOW' AND remote_login = N'lnk_from_sql01' THEN N'PASS'
             WHEN expected = N'DENY'  AND actual = N'DENY'  AND detail LIKE N'7416:%'           THEN N'PASS'
             ELSE N'FAIL' END AS result
FROM @Results;

IF @HasDefaultMapping = 1
    PRINT 'Note: a mapping for <all logins> exists, so any login on SQL01 can use SQL02. Do Step 6 if that was not intended.';

DROP LOGIN lnk_test_denied;
```

Notes:

- If AppLogin is a Windows login, use the full name, for example `N'DOMAIN\svc_app'`. It must exist as a login on SQL01
- If you would rather not create a login, replace `lnk_test_denied` with any existing low-privilege login that has no mapping, and remove the CREATE and DROP lines
- Impersonation only proves a **mapped remote login** setup. If you chose "Be made using the login's current security context" (Kerberos), impersonated tests fail by design; test that model by connecting as a real user from the application server
- Last check: ask the application team to run their actual query from the application before you close the change

<!-- Screenshot: save as docs/images/linked-server/step-10.png and uncomment the next line -->
<!-- ![Step 10: Impersonation testing before handoff](../images/linked-server/step-10.png) -->

> [!TIP]
> **Expected result:** Both rows show PASS. AppLogin is ALLOW with remote_login lnk_from_sql01, and lnk_test_denied is DENY with error 7416 (Access to the remote server is denied because no login-mapping exists).

## Step 11: Script it out and hand off

In SSMS right-click the linked server > **Script Linked Server as** > **CREATE To** > **New Query Editor Window**. Save the script (remove the password first) with the change record. Then list what exists on the server:

```sql
SELECT s.name AS linked_server, s.provider, s.data_source, s.is_rpc_out_enabled, s.is_data_access_enabled,
       l.uses_self_credential, l.remote_name AS remote_login, SUSER_NAME(l.local_principal_id) AS local_login
FROM sys.servers AS s
LEFT JOIN sys.linked_logins AS l ON l.server_id = s.server_id
WHERE s.is_linked = 1;
```

Attach these to the ServiceNow change as handoff evidence:

- The CREATE script (no password)
- The output of the Step 9 validation script and the Step 10 impersonation test
- A short note for the application team: the linked server name, how to call it (`SQL02.YourDatabase.dbo.YourTable` or `OPENQUERY(SQL02, '...')`), what it can and cannot do (read-only, no remote procedures), and who to contact

<!-- Screenshot: save as docs/images/linked-server/step-11.png and uncomment the next line -->
<!-- ![Step 11: Script it out and hand off](../images/linked-server/step-11.png) -->

> [!TIP]
> **Expected result:** SQL02 is listed with the provider, data source, options and login mapping you intended, and the evidence is attached to the change.

## Step 12: Roll back if needed

To remove the linked server and its login mappings from SQL01, then the login from SQL02:

```sql
-- On SQL01
EXEC master.dbo.sp_dropserver N'SQL02', 'droplogins';

-- On SQL02
USE YourDatabase; DROP USER lnk_from_sql01;
DROP LOGIN lnk_from_sql01;
```

<!-- Screenshot: save as docs/images/linked-server/step-12.png and uncomment the next line -->
<!-- ![Step 12: Roll back if needed](../images/linked-server/step-12.png) -->

> [!TIP]
> **Expected result:** SQL02 no longer appears under Linked Servers, and the login is gone from SQL02.

## Validation sign-off

- [ ] Remote login on SQL02 is dedicated, not sa or a personal account, and has only the access needed
- [ ] sp_testlinkedserver succeeds from SQL01
- [ ] A four-part query and an OPENQUERY both return rows
- [ ] OPENQUERY shows the expected remote login
- [ ] RPC Out is on only if remote procedures are required
- [ ] Distributed transaction promotion is off unless required
- [ ] Only the intended local logins can use the link
- [ ] Encryption is on, and any TrustServerCertificate exception is approved
- [ ] Validation script (Step 9): every check PASS, or REVIEW items explained in the change
- [ ] Impersonation test (Step 10): AppLogin allowed as lnk_from_sql01, unmapped login refused with error 7416
- [ ] Throwaway test login dropped
- [ ] Application team ran their real query successfully
- [ ] CREATE script (without the password) and test outputs attached to the change

## Sources

- **Microsoft Learn:** [Create linked servers (SQL Server Database Engine)](https://msdn.microsoft.com/library/ff772782.aspx)
- **Microsoft Learn:** [Linked servers (Database Engine)](https://learn.microsoft.com/en-us/sql/relational-databases/linked-servers/linked-servers-database-engine)
- **Microsoft Learn:** [sp_testlinkedserver](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-testlinkedserver-transact-sql)
- **Microsoft Learn:** [EXECUTE AS (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/execute-as-transact-sql)
- **Microsoft Learn:** [CONNECTIONPROPERTY (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/functions/connectionproperty-transact-sql)
- **MSSQLTips:** [Understanding SQL Server Linked Servers](https://www.mssqltips.com/sqlservertip/6083/understanding-sql-server-linked-servers/)
- **SQLShack:** [How to create and configure a linked server in SQL Server Management Studio](https://www.sqlshack.com/how-to-create-and-configure-a-linked-server-in-sql-server-management-studio/)

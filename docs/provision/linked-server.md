# Create a linked server (SQL Server to SQL Server)

**Status:** Draft · **Applies to:** SQL Server 2016 to 2022 · **Steps:** 10

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

## Step 9: Script it out and document it

In SSMS right-click the linked server > **Script Linked Server as** > **CREATE To** > **New Query Editor Window**. Save the script (remove the password first) with the change record. Then list what exists on the server:

```sql
SELECT s.name AS linked_server, s.provider, s.data_source, s.is_rpc_out_enabled, s.is_data_access_enabled,
       l.uses_self_credential, l.remote_name AS remote_login, SUSER_NAME(l.local_principal_id) AS local_login
FROM sys.servers AS s
LEFT JOIN sys.linked_logins AS l ON l.server_id = s.server_id
WHERE s.is_linked = 1;
```

<!-- Screenshot: save as docs/images/linked-server/step-09.png and uncomment the next line -->
<!-- ![Step 9: Script it out and document it](../images/linked-server/step-09.png) -->

> [!TIP]
> **Expected result:** SQL02 is listed with the provider, data source, options and login mapping you intended.

## Step 10: Roll back if needed

To remove the linked server and its login mappings from SQL01, then the login from SQL02:

```sql
-- On SQL01
EXEC master.dbo.sp_dropserver N'SQL02', 'droplogins';

-- On SQL02
USE YourDatabase; DROP USER lnk_from_sql01;
DROP LOGIN lnk_from_sql01;
```

<!-- Screenshot: save as docs/images/linked-server/step-10.png and uncomment the next line -->
<!-- ![Step 10: Roll back if needed](../images/linked-server/step-10.png) -->

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
- [ ] CREATE script saved (without the password) in the change record

## Sources

- **Microsoft Learn:** [Create linked servers (SQL Server Database Engine)](https://msdn.microsoft.com/library/ff772782.aspx)
- **Microsoft Learn:** [Linked servers (Database Engine)](https://learn.microsoft.com/en-us/sql/relational-databases/linked-servers/linked-servers-database-engine)
- **MSSQLTips:** [Understanding SQL Server Linked Servers](https://www.mssqltips.com/sqlservertip/6083/understanding-sql-server-linked-servers/)
- **SQLShack:** [How to create and configure a linked server in SQL Server Management Studio](https://www.sqlshack.com/how-to-create-and-configure-a-linked-server-in-sql-server-management-studio/)

# Install SQL Server 2022 (standalone instance)

**Status:** Draft · **Applies to:** SQL Server 2022 on Windows Server 2019 / 2022 · **Steps:** 11

Build a standalone SQL Server 2022 Database Engine instance through the Setup wizard, with service accounts, drive layout, tempdb and memory set correctly during install, then patch it to the latest cumulative update.

> [!NOTE]
> **Before you start**
>
> - Windows Server patched and joined to the domain, with a pending-reboot check done
> - Service accounts created for the Engine and Agent (gMSA preferred, otherwise domain accounts with non-expiring passwords set by policy)
> - Separate volumes for data, log, tempdb and backups, formatted NTFS or ReFS with a 64 KB allocation unit (for example D:, L:, T:, X:)
> - SQL Server 2022 installation media (ISO) and the latest cumulative update downloaded
> - An AD group for DBAs to add as sysadmin
> - Local administrator rights on the server, and a change ticket if this is not a lab

## Step 1: Check the server is ready

Run in an elevated PowerShell window. Confirm the OS version, that the SQL volumes exist with a **64 KB allocation unit**, and that no reboot is pending.

```powershell
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, LastBootUpTime

Get-Volume | Where-Object DriveLetter |
    Select-Object DriveLetter, FileSystemLabel, FileSystem, AllocationUnitSize,
        @{n='SizeGB';e={[math]::Round($_.Size/1GB,1)}},
        @{n='FreeGB';e={[math]::Round($_.SizeRemaining/1GB,1)}}

# Any of these existing means a reboot is pending
Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending'
Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired'
```

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-01.png and uncomment the next line -->
<!-- ![Step 1: Check the server is ready](../images/install-sql-server-2022/step-01.png) -->

> [!TIP]
> **Expected result:** AllocationUnitSize is 65536 on the data, log and tempdb volumes, and both Test-Path lines return False.

## Step 2: Start Setup

Mount the ISO and run `setup.exe` as administrator. In the SQL Server Installation Center choose **Installation** > **New SQL Server standalone installation or add features to an existing installation**.

- Enter the product key or choose the edition (Developer or Evaluation for a lab)
- Accept the license terms
- Tick **Use Microsoft Update to check for updates** only if your patching process allows it

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-02.png and uncomment the next line -->
<!-- ![Step 2: Start Setup](../images/install-sql-server-2022/step-02.png) -->

> [!TIP]
> **Expected result:** The Install Rules page shows no failures. A Windows Firewall warning is normal; you open the port after install.

## Step 3: Skip the Azure extension unless you need it

SQL Server 2022 shows an **Azure Extension for SQL Server** page. Untick it unless this server is being onboarded to Azure Arc as part of the build. It can be added later.

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-03.png and uncomment the next line -->
<!-- ![Step 3: Skip the Azure extension unless you need it](../images/install-sql-server-2022/step-03.png) -->

## Step 4: Select features

Tick **Database Engine Services** only. Add **Full-Text and Semantic Extractions for Search** only if the application needs it. Leave Machine Learning Services, PolyBase and the other features unticked so you do not install components nobody patches or monitors.

Keep the shared feature and instance root directories on C: (binaries only).

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-04.png and uncomment the next line -->
<!-- ![Step 4: Select features](../images/install-sql-server-2022/step-04.png) -->

> [!TIP]
> **Expected result:** Only the features the application needs are selected.

## Step 5: Name the instance

Choose **Default instance** (MSSQLSERVER) unless your standard uses named instances. For a named instance, note the name; you will set a static port for it after install.

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-05.png and uncomment the next line -->
<!-- ![Step 5: Name the instance](../images/install-sql-server-2022/step-05.png) -->

## Step 6: Set service accounts, startup and IFI

On **Server Configuration**:

- Set **SQL Server Database Engine** and **SQL Server Agent** to their own service accounts (gMSA accounts end in `$` and have no password)
- Set both startup types to **Automatic**
- Tick **Grant Perform Volume Maintenance Task privilege to SQL Server Database Engine Service**. This enables instant file initialization so data file growth and restores are fast
- On the **Collation** tab, keep the default (`SQL_Latin1_General_CP1_CI_AS` on an English US server) unless the application vendor specifies another. Collation is hard to change later

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-06.png and uncomment the next line -->
<!-- ![Step 6: Set service accounts, startup and IFI](../images/install-sql-server-2022/step-06.png) -->

> [!TIP]
> **Expected result:** Engine and Agent use separate domain or gMSA accounts, Agent is Automatic, and the IFI box is ticked.

## Step 7: Configure the Database Engine

On **Database Engine Configuration**, work through each tab:

- **Server Configuration**: Windows authentication mode unless the application needs SQL logins (then Mixed Mode with a strong sa password). Click **Add** and add the DBA AD group as sysadmin. Avoid adding individual people
- **Data Directories**: user database data on `D:\SQLData`, logs on `L:\SQLLogs`, backups on `X:\SQLBackups`
- **TempDB**: one data file per logical core up to 8, all the same size and growth, data and log on `T:\TempDB`
- **MaxDOP**: accept the recommended value Setup calculates
- **Memory**: choose **Recommended** and tick the box to accept it
- **FILESTREAM**: leave off unless required

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-07.png and uncomment the next line -->
<!-- ![Step 7: Configure the Database Engine](../images/install-sql-server-2022/step-07.png) -->

> [!TIP]
> **Expected result:** Every tab reflects the drive layout and the DBA group appears in the sysadmin list.

## Step 8: Save the configuration file and install

On **Ready to Install**, note the path to **ConfigurationFile.ini** and copy it somewhere safe. It lets you repeat this exact install unattended on the next server. Then click **Install**.

To reuse it later, remove the line `UIMODE` if present, add `IACCEPTSQLSERVERLICENSETERMS="True"` and run:

```bat
setup.exe /ConfigurationFile="C:\Install\ConfigurationFile.ini" /SAPWD="<only if Mixed Mode>" /Q
```

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-08.png and uncomment the next line -->
<!-- ![Step 8: Save the configuration file and install](../images/install-sql-server-2022/step-08.png) -->

> [!TIP]
> **Expected result:** Setup finishes with every feature showing Succeeded. Check the Summary.txt log if anything shows Failed.

## Step 9: Patch to the latest cumulative update

Run the latest SQL Server 2022 CU you downloaded. Accept the defaults; it patches the instance and shared components. Reboot if Setup asks for it.

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-09.png and uncomment the next line -->
<!-- ![Step 9: Patch to the latest cumulative update](../images/install-sql-server-2022/step-09.png) -->

> [!TIP]
> **Expected result:** The CU installer reports success for the instance.

## Step 10: Confirm the build

Connect in SSMS and run:

```sql
SELECT @@SERVERNAME                                  AS server_name,
       SERVERPROPERTY('ProductVersion')              AS product_version,
       SERVERPROPERTY('ProductUpdateLevel')          AS cumulative_update,
       SERVERPROPERTY('Edition')                     AS edition,
       SERVERPROPERTY('Collation')                   AS collation,
       SERVERPROPERTY('IsIntegratedSecurityOnly')    AS windows_auth_only;

SELECT servicename, service_account, startup_type_desc, status_desc,
       instant_file_initialization_enabled
FROM sys.dm_server_services;
```

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-10.png and uncomment the next line -->
<!-- ![Step 10: Confirm the build](../images/install-sql-server-2022/step-10.png) -->

> [!TIP]
> **Expected result:** cumulative_update shows the CU you installed, both services are Running and Automatic, and instant_file_initialization_enabled is Y for the Engine.

## Step 11: Open the firewall port

Allow inbound TCP 1433 for a default instance (or the static port you assign to a named instance). Restrict the rule to the networks that need it if your standard requires it.

```powershell
New-NetFirewallRule -DisplayName 'SQL Server 1433' -Direction Inbound -Protocol TCP -LocalPort 1433 -Action Allow -Profile Domain
```

<!-- Screenshot: save as docs/images/install-sql-server-2022/step-11.png and uncomment the next line -->
<!-- ![Step 11: Open the firewall port](../images/install-sql-server-2022/step-11.png) -->

> [!TIP]
> **Expected result:** From an application or jump server, Test-NetConnection <server> -Port 1433 shows TcpTestSucceeded : True.

## Validation sign-off

- [ ] SELECT @@VERSION shows SQL Server 2022 at the latest CU
- [ ] Engine and Agent run as their own service accounts, startup Automatic
- [ ] Instant file initialization is Y in sys.dm_server_services
- [ ] tempdb has equal-sized data files, one per core up to 8, on the tempdb volume
- [ ] Default data, log and backup paths point to the right volumes
- [ ] The DBA AD group is sysadmin and no individual accounts are
- [ ] ConfigurationFile.ini saved with the build notes
- [ ] Port reachable from the application network

## Sources

- **Microsoft Learn:** [Install SQL Server from the Installation Wizard (Setup)](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/install-sql-server-from-the-installation-wizard-setup)
- **Microsoft Learn:** [Install SQL Server using a configuration file](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/install-sql-server-using-a-configuration-file)
- **MSSQLTips:** [How to install SQL Server 2022 step by step](https://www.mssqltips.com/sqlservertip/7313/install-sql-server-2022/)
- **MSSQLTips:** [SQL Server Pre SQL Installation Checklist (Part 3 of 4)](https://www.mssqltips.com/sqlservertip/2283/sql-server-pre-sql-installation-checklist-part-3-of-4/)
- **Microsoft Learn:** [Database instant file initialization](https://learn.microsoft.com/en-us/sql/relational-databases/databases/database-instant-file-initialization)

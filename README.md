# SQL Server Docs

Step-by-step SQL Server runbooks for **installation, configuration, provisioning,
setup and validation**, from the first install screen to the final sign-off.

Every runbook lists what you need before you start, the steps in order, the
commands to run, what you should see when a step has worked, a sign-off
checklist, and the Microsoft Learn, MSSQLTips and SQLShack articles behind it.

## The build, in order

| Phase | What it covers | Runbooks |
|---|---|---|
| **1. [Install](docs/install/)** | Pre-install checks, Setup, service accounts, patching, upgrades | [Install SQL Server 2022 (standalone instance)](docs/install/install-sql-server-2022.md)<br>[In-place upgrade: SQL Server 2019 to 2022](docs/install/upgrade-sql-server-2019-to-2022.md)<br>[Upgrade SSMS 21 / 22 in place](docs/install/upgrade-ssms-22.md) |
| **2. [Configure](docs/configure/)** | Memory, MAXDOP, tempdb, defaults, error logs, alerts | [Post-install configuration baseline](docs/configure/post-install-baseline.md) |
| **3. [Provision](docs/provision/)** | Databases, logins, security, connections between instances | [Create a linked server](docs/provision/linked-server.md)<br>[Refresh a non-production database](docs/provision/database-refresh.md) |
| **4. [Setup](docs/setup/)** | HA/DR, backups, maintenance, monitoring, Agent jobs | Coming next |
| **5. [Validate](docs/validate/)** | Health checks and sign-off before handover | [New instance validation and sign-off](docs/validate/new-instance-validation.md) |

## How to use a runbook

1. Read **Before you start** and make sure every item is true.
2. Work through the steps in order. Use the copy button on each code block.
3. Compare what you see with the **Expected result** under each step.
4. Tick off the **Validation sign-off** list and keep the outputs with your change record.

> [!TIP]
> **Status labels:** *Draft* means written but not yet run end to end. *Reviewed* means checked by a second DBA.
> *Tested on Dev* means it has been run for real on a Dev server. *Tested in lab* means every step has been run and screenshotted on a lab build.

## Adding runbooks and screenshots

See [Contributing](docs/contributing/) and copy the [runbook template](docs/contributing/runbook-template.md).

## Scripts

The T-SQL, PowerShell and Robocopy scripts these runbooks refer to live in
[MySqlScripts](https://github.com/Aderemil28/MySqlScripts).

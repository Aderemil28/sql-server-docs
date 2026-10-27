# Copy refresh backups in parallel batches with Robocopy

**Status:** Draft · **Applies to:** Windows Server 2016 and later, PowerShell 5.1 and later · **Steps:** 8

Copy the full backups of many databases from the production backup share to a refresh server before a multi-database refresh. Small databases are copied a few at a time in parallel, and the largest ones are copied on their own so they get the full bandwidth. Every copy is logged and checked, so you know exactly which databases are ready to restore.

Use this with [Refresh a non-production database from production](database-refresh.md): it replaces that runbook's single-file copy when a refresh covers many databases.

In this runbook **BACKUPSRV01** is the server holding the backup share, **SQLCLUSTER01$AG01** is the backup folder for the source availability group, and **H:\Refresh** is the local folder on the refresh server. Database names such as **AppDB01** are examples. Replace these with your own names.

> [!NOTE]
> **Before you start**
>
> - The refresh request is approved and lists every database to copy (see Step 1 of the refresh runbook)
> - The account running the copy can read the backup share and write to the destination folder
> - The backups follow a `<server or AG folder>\<database>\FULL\` layout, as Ola Hallengren's backup jobs create. If yours differ, change the paths in Step 4
> - A PowerShell window **run as administrator** on the refresh server
> - Agreement with the storage or network team if you are copying many terabytes in business hours

> [!WARNING]
> These files contain **unmasked production data**. Copy only to approved storage, and delete them once the refresh is signed off.

## Step 1: List the databases and their backup sizes

Size decides the batching. On the source primary, run [largest-databases.sql](https://github.com/Aderemil28/MySqlScripts/blob/main/tsql/storage/largest-databases.sql), or check the backup sizes on the share:

```powershell
$share = '\\BACKUPSRV01\SQL_Backup\SQLCLUSTER01$AG01'
Get-ChildItem $share -Directory | ForEach-Object {
    $full = Join-Path $_.FullName 'FULL'
    $gb = (Get-ChildItem $full -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum / 1GB
    [pscustomobject]@{ Database = $_.Name; FullBackupGB = [math]::Round($gb, 1) }
} | Sort-Object FullBackupGB -Descending | Format-Table -AutoSize
```

<!-- Screenshot: save as docs/images/copy-refresh-backups-in-batches/step-01.png and uncomment the next line -->
<!-- ![Step 1: List the databases and their backup sizes](../images/copy-refresh-backups-in-batches/step-01.png) -->

> [!TIP]
> **Expected result:** A list of every database in the request with its full backup size.

## Step 2: Check there is room on the destination

Add up the backup sizes from Step 1, then compare with the free space on the destination drive. Leave at least 10% spare.

```powershell
Get-Volume -DriveLetter H | Select-Object DriveLetter,
    @{N='SizeGB';E={[math]::Round($_.Size/1GB,1)}},
    @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,1)}}
```

<!-- Screenshot: save as docs/images/copy-refresh-backups-in-batches/step-02.png and uncomment the next line -->
<!-- ![Step 2: Check there is room on the destination](../images/copy-refresh-backups-in-batches/step-02.png) -->

> [!TIP]
> **Expected result:** FreeGB is larger than the total backup size plus 10%.

## Step 3: Plan the batches

Group the databases so each batch finishes in a similar time:

| Group | Rule of thumb | Run as |
|---|---|---|
| Small and medium | Under about 50 GB each | Batches of 4, in parallel |
| Large | About 50 GB and over | One at a time, on its own |

Four copies in parallel, each with `/MT:16`, is usually enough to fill a 10 Gb link. More parallel copies rarely makes it faster and can slow the source share for production backups.

<!-- Screenshot: save as docs/images/copy-refresh-backups-in-batches/step-03.png and uncomment the next line -->
<!-- ![Step 3: Plan the batches](../images/copy-refresh-backups-in-batches/step-03.png) -->

> [!TIP]
> **Expected result:** Every database in the request is in exactly one batch.

## Step 4: Set the paths and batches

Paste this into the PowerShell window and change the values to your own.

```powershell
$source  = '\\BACKUPSRV01\SQL_Backup\SQLCLUSTER01$AG01'   # single quotes: $AG01 is part of the folder name
$dest    = 'H:\Refresh'
$logDir  = 'H:\Refresh\_logs'

$batches = [ordered]@{
    'Batch 1 - smallest' = @('AppDB01', 'AppDB02', 'AppDB03', 'AppDB04')
    'Batch 2'            = @('AppDB05', 'AppDB06', 'AppDB07', 'AppDB08')
    'Batch 3'            = @('AppDB09', 'AppDB10', 'AppDB11', 'AppDB12')
    'Large - AppDB13'    = @('AppDB13')    # for example 76 GB
    'Large - AppDB14'    = @('AppDB14')    # for example 324 GB
    'Large - AppDB15'    = @('AppDB15')    # for example 439 GB
}

New-Item -ItemType Directory -Path $dest, $logDir -Force | Out-Null
```

> [!IMPORTANT]
> Folder names created by Ola Hallengren's jobs contain a `$` (`SERVER$INSTANCE` or `CLUSTER$AG`). In **double** quotes PowerShell treats `$AG01` as a variable and drops it, so the path silently becomes wrong. Use single quotes as above, or escape it with a backtick in double quotes: ``"\\BACKUPSRV01\SQL_Backup\SQLCLUSTER01`$AG01"``.

<!-- Screenshot: save as docs/images/copy-refresh-backups-in-batches/step-04.png and uncomment the next line -->
<!-- ![Step 4: Set the paths and batches](../images/copy-refresh-backups-in-batches/step-04.png) -->

> [!TIP]
> **Expected result:** `Test-Path "$source\AppDB01\FULL"` returns **True** for a database in the list.

## Step 5: Run the copy

This runs each batch's databases in parallel, waits for the batch to finish, then starts the next. Every database gets its own log and its Robocopy exit code is captured.

```powershell
$results = foreach ($batchName in $batches.Keys) {
    Write-Host "Starting $batchName" -ForegroundColor Cyan
    $jobs = foreach ($db in $batches[$batchName]) {
        Start-Job -Name $db -ArgumentList $source, $dest, $logDir, $db -ScriptBlock {
            param($src, $dst, $logs, $dbname)
            $log = Join-Path $logs "$dbname.log"
            robocopy "$src\$dbname\FULL" "$dst\$dbname\FULL" *.* /MT:16 /J /XJ /R:2 /W:5 /NP /XX /LOG:$log | Out-Null
            [pscustomobject]@{ Database = $dbname; ExitCode = $LASTEXITCODE; Log = $log }
        }
    }
    $jobs | Wait-Job | Receive-Job | Select-Object Database, ExitCode, Log
    $jobs | Remove-Job
}
$results | Format-Table -AutoSize
```

What the switches do:

| Switch | Why |
|---|---|
| `/MT:16` | 16 threads per copy. Helps most for striped backups (many files per database) |
| `/J` | Unbuffered I/O, faster and lighter on memory for large backup files |
| `/XJ` | Skip junction points, so a link in the folder can't send the copy somewhere unexpected |
| `/R:2 /W:5` | Retry a failed file twice, 5 seconds apart. The default is a million retries, and `/R:0` gives up on the first network blip |
| `/NP /LOG:` | No progress percentage in the log, one log per database |
| `/XX` | Don't list files that exist only in the destination. Nothing is deleted either way |

<!-- Screenshot: save as docs/images/copy-refresh-backups-in-batches/step-05.png and uncomment the next line -->
<!-- ![Step 5: Run the copy](../images/copy-refresh-backups-in-batches/step-05.png) -->

> [!TIP]
> **Expected result:** One row per database, every ExitCode below 8.

## Step 6: Check the exit codes

Robocopy exit codes below 8 are success: 0 means nothing needed copying, 1 means files were copied, 2 and 3 mean extra files exist in the destination. 8 or higher means at least one file failed.

```powershell
$failed = $results | Where-Object ExitCode -ge 8
if ($failed) {
    $failed | Format-Table -AutoSize
    $failed | ForEach-Object { Get-Content $_.Log -Tail 20 }
} else {
    Write-Host 'All copies succeeded' -ForegroundColor Green
}
```

To re-run only the failed ones, set `$batches = [ordered]@{ 'Retry' = @($failed.Database) }` and run Step 5 again. Robocopy skips files that are already complete.

<!-- Screenshot: save as docs/images/copy-refresh-backups-in-batches/step-06.png and uncomment the next line -->
<!-- ![Step 6: Check the exit codes](../images/copy-refresh-backups-in-batches/step-06.png) -->

> [!TIP]
> **Expected result:** *All copies succeeded*, or a short list you have re-run until it is empty.

## Step 7: Compare source and destination

Exit codes prove Robocopy finished, not that you copied the right backup. Compare file counts and sizes:

```powershell
$compare = foreach ($db in ($batches.Values | ForEach-Object { $_ })) {
    $s = Get-ChildItem "$source\$db\FULL" -File | Measure-Object Length -Sum
    $d = Get-ChildItem "$dest\$db\FULL"   -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum
    [pscustomobject]@{
        Database = $db
        SourceFiles = $s.Count; DestFiles = $d.Count
        SourceGB = [math]::Round($s.Sum/1GB,2); DestGB = [math]::Round($d.Sum/1GB,2)
        Match = ($s.Count -eq $d.Count -and $s.Sum -eq $d.Sum)
    }
}
$compare | Format-Table -AutoSize
```

Then check the newest backup in each folder is from the date the request asked for, and run `RESTORE VERIFYONLY` on it as in the refresh runbook.

<!-- Screenshot: save as docs/images/copy-refresh-backups-in-batches/step-07.png and uncomment the next line -->
<!-- ![Step 7: Compare source and destination](../images/copy-refresh-backups-in-batches/step-07.png) -->

> [!TIP]
> **Expected result:** Match is **True** for every database.

## Step 8: Record the evidence

Attach to the ticket: the `$results` table from Step 5, the `$compare` table from Step 7, and the `_logs` folder (zipped). Then carry on with the restore steps in the refresh runbook. Delete the copied backups once the refresh is signed off.

<!-- Screenshot: save as docs/images/copy-refresh-backups-in-batches/step-08.png and uncomment the next line -->
<!-- ![Step 8: Record the evidence](../images/copy-refresh-backups-in-batches/step-08.png) -->

> [!TIP]
> **Expected result:** The ticket holds the copy results, the comparison and the logs.

## Validation sign-off

- [ ] Every database in the request is in one batch
- [ ] Destination had enough free space before the copy
- [ ] Source path checked with Test-Path (no lost `$` in the folder name)
- [ ] Every Robocopy exit code below 8
- [ ] File counts and sizes match for every database
- [ ] Newest backup in each folder is from the requested date
- [ ] Results, comparison and logs attached to the ticket
- [ ] Copies scheduled for deletion after sign-off

## Sources

- **Microsoft Learn:** [robocopy](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy)
- **Microsoft Learn:** [Start-Job](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/start-job)
- **Microsoft Learn:** [about_Quoting_Rules](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_quoting_rules)
- **Ola Hallengren:** [SQL Server Backup](https://ola.hallengren.com/sql-server-backup.html)
- **Toolkit:** [Copy-BackupFiles.ps1](https://github.com/Aderemil28/MySqlScripts/blob/main/robocopy/Copy-BackupFiles.ps1) and [robocopy-cheatsheet.md](https://github.com/Aderemil28/MySqlScripts/blob/main/robocopy/robocopy-cheatsheet.md)

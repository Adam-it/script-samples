

# Restore large amount of items from SharePoint Recycle bin in bulk

## Summary

Restores items from the recycle bin based on its unique ID, a GUID.  
Attempts in batches of "x" items (default 10), if a failure occurs, not all items are restored, therefore will attempt to restore the items in the batch individually before grabbing the next batch. The results of the restore can be saved to a CSV file. You can choose between PnP PowerShell and CLI for Microsoft 365 versions depending on the tooling available in your environment. The CLI option also supports restoring every item from the first- or second-stage recycle bin without preparing a CSV.

Script allows to restore in batches of 100 if you wish, however, if failures are found it could take a longer overall, as script falls back to restoring each item individually to ensure all are restored and report the error item(s).
### Prerequisites

- Obtained details of the items to restore from the sites recycle bin in a csv file.
  ```powershell
      ## PnP PowerShell
      Connect-PnPOnline -url:https://contso.sharepoint.com/sites/RestoreDocs -pnpManagementShell
      $recycleBinItems = Get-PnpRecycleBinItem -FirstStage -RowLimit 999999
      $recycleBinItems | Export-Csv .\recyclebin.csv -NoTypeInformation

      ## CLI for Microsoft 365
      m365 login --ensure
      m365 spo site recyclebinitem list --siteUrl https://contso.sharepoint.com/sites/RestoreDocs --output csv > ./recyclebin.csv
  ```
- Open csv and remove rows you do not wish to restore. Save the csv file.

### Screenshots
Screen Output

![Screen Output](assets/screen-output.png)

CSV Output

![CSV Output](assets/csv-output.png)


# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
function Restore-SpoRecycleBinItems {
    [CmdletBinding(SupportsShouldProcess = $true, DefaultParameterSetName = 'Ids')]
    param(
        [Parameter(Mandatory = $true, HelpMessage = 'Absolute URL of the SharePoint site collection that owns the recycle bin.')]
        [ValidateNotNullOrEmpty()]
        [Uri]
        $SiteUrl,

        [Parameter(Mandatory = $true, ParameterSetName = 'Ids', HelpMessage = 'Path to the CSV file containing recycle bin entries exported earlier.')]
        [ValidateNotNullOrEmpty()]
        [string]
        $InputCsvPath,

        [Parameter(ParameterSetName = 'Ids', HelpMessage = 'Destination CSV file that will store the restore results.')]
        [string]
        $OutputCsvPath,

        [Parameter(ParameterSetName = 'Ids', HelpMessage = 'Number of recycle bin items to restore in a single batch.')]
        [ValidateRange(1, 200)]
        [int]
        $BatchSize = 10,

        [Parameter(ParameterSetName = 'All', HelpMessage = 'Restore all items from the first-stage recycle bin.')]
        [switch]
        $AllPrimary,

        [Parameter(ParameterSetName = 'All', HelpMessage = 'Restore all items from the second-stage recycle bin.')]
        [switch]
        $AllSecondary
    )

    begin {
        Write-Verbose 'Ensuring CLI for Microsoft 365 session.'
        m365 login --ensure
        if ($LASTEXITCODE -ne 0) {
            throw 'm365 login failed. Please authenticate before running the script.'
        }

        $summary = [ordered]@{
            SiteUrl          = $SiteUrl.AbsoluteUri
            Mode             = if ($PSCmdlet.ParameterSetName -eq 'Ids') { 'CsvIds' } else { 'RecycleBinScope' }
            Scope            = $null
            TotalItems       = $null
            BatchSize        = $null
            BatchesAttempted = $null
            Restored         = $null
            Skipped          = $null
            Failed           = $null
            OperationStatus  = 'Pending'
            Results          = @()
            Failures         = @()
        }

        if ($PSCmdlet.ParameterSetName -eq 'Ids') {
            Write-Verbose 'Validating input CSV path.'
            $resolvedInput = Resolve-Path -Path $InputCsvPath -ErrorAction Stop
            $items = @(Import-Csv -Path $resolvedInput.Path)
            if ($items.Count -eq 0) {
                throw "Input file '$($resolvedInput.Path)' does not contain any entries."
            }

            if (-not $OutputCsvPath) {
                $OutputCsvPath = Join-Path -Path (Split-Path -Parent $resolvedInput.Path) -ChildPath 'recyclebin-results.csv'
            }

            $summary.Scope            = 'IdsFromCsv'
            $summary.TotalItems       = $items.Count
            $summary.BatchSize        = $BatchSize
            $summary.BatchesAttempted = 0
            $summary.Restored         = 0
            $summary.Skipped          = 0
            $summary.Failed           = 0

            $script:ItemsToRestore    = $items
            $script:OutputCsvPath     = $OutputCsvPath
            $script:TotalBatches      = [Math]::Ceiling($items.Count / [double]$BatchSize)
        }
        else {
            if (-not ($AllPrimary -or $AllSecondary)) {
                throw 'Specify at least one of -AllPrimary or -AllSecondary when using the recycle bin scope mode.'
            }

            $summary.Scope = if ($AllPrimary -and $AllSecondary) {
                'Primary and Secondary'
            }
            elseif ($AllPrimary) {
                'Primary'
            }
            else {
                'Secondary'
            }

            $script:OutputCsvPath = $null
        }

        $script:Summary          = $summary
        $script:ParameterSetName = $PSCmdlet.ParameterSetName
    }

    process {
        if ($script:ParameterSetName -eq 'Ids') {
            $items        = $script:ItemsToRestore
            $totalBatches = [Math]::Ceiling($items.Count / [double]$BatchSize)

            for ($index = 0; $index -lt $items.Count; $index += $BatchSize) {
                $endIndex = [Math]::Min($index + $BatchSize - 1, $items.Count - 1)
                $batch    = $items[$index..$endIndex]
                if ($batch -isnot [System.Array]) {
                    $batch = @($batch)
                }

                $batchNumber = [int]([Math]::Floor($index / $BatchSize) + 1)
                Write-Verbose ("Processing batch {0}/{1} containing {2} item(s)." -f $batchNumber, $totalBatches, $batch.Count)

                $batchIds       = @($batch | ForEach-Object { $_.Id })
                $missingEntries = $batch | Where-Object { -not $_.Id }
                foreach ($entry in $missingEntries) {
                    $script:Summary.Skipped++
                    $script:Summary.Results += [pscustomobject]@{
                        Id      = $entry.Id
                        Title   = $entry.Title
                        Status  = 'Skipped'
                        Message = 'Missing Id value in source CSV.'
                    }
                }

                $batch   = $batch | Where-Object { $_.Id }
                $batchIds = @($batch | ForEach-Object { $_.Id })
                if ($batch.Count -eq 0) {
                    continue
                }

                if (-not $PSCmdlet.ShouldProcess($SiteUrl.AbsoluteUri, "Restore $($batch.Count) recycle bin item(s)")) {
                    $script:Summary.OperationStatus = 'WhatIf'
                    foreach ($entry in $batch) {
                        $script:Summary.Skipped++
                        $script:Summary.Results += [pscustomobject]@{
                            Id      = $entry.Id
                            Title   = $entry.Title
                            Status  = 'WhatIf'
                            Message = 'Operation skipped due to WhatIf preference.'
                        }
                    }
                    continue
                }

                $idsParameter = $batchIds -join ','
                Write-Verbose ("Restoring batch {0}/{1}." -f $batchNumber, $totalBatches)
                $batchRestoreOutput = m365 spo site recyclebinitem restore --siteUrl $SiteUrl.AbsoluteUri --ids "$idsParameter" 2>&1

                if ($LASTEXITCODE -eq 0) {
                    $script:Summary.Restored += $batch.Count
                    foreach ($entry in $batch) {
                        $script:Summary.Results += [pscustomobject]@{
                            Id      = $entry.Id
                            Title   = $entry.Title
                            Status  = 'Restored'
                            Message = 'Restored in batch.'
                        }
                    }
                }
                else {
                    $failureMessage = if ([string]::IsNullOrWhiteSpace($batchRestoreOutput)) {
                        'Restore failed. See CLI output for details.'
                    }
                    else {
                        $batchRestoreOutput.ToString()
                    }

                    $script:Summary.Failed += $batch.Count
                    foreach ($entry in $batch) {
                        $script:Summary.Results += [pscustomobject]@{
                            Id      = $entry.Id
                            Title   = $entry.Title
                            Status  = 'Failed'
                            Message = $failureMessage
                        }
                    }

                    $script:Summary.Failures += [pscustomobject]@{
                        Stage   = 'BatchRestore'
                        Target  = $idsParameter
                        Message = $failureMessage
                    }

                    Write-Warning ("Failed to restore batch {0}/{1}. CLI output: {2}" -f $batchNumber, $totalBatches, $failureMessage)
                }

                $script:Summary.BatchesAttempted++
            }

            if ($script:Summary.OperationStatus -eq 'Pending') {
                $script:Summary.OperationStatus = if ($script:Summary.Failed -gt 0) { 'CompletedWithErrors' } else { 'Completed' }
            }
        }
        else {
            $scopeLabel = $script:Summary.Scope
            if (-not $PSCmdlet.ShouldProcess($SiteUrl.AbsoluteUri, "Restore recycle bin items from $scopeLabel")) {
                $script:Summary.OperationStatus = 'WhatIf'
                $script:Summary.Results += [pscustomobject]@{
                    Scope   = $scopeLabel
                    Status  = 'WhatIf'
                    Message = 'Operation skipped due to WhatIf preference.'
                }
                return
            }

            Write-Verbose "Restoring recycle bin scope '$scopeLabel'."
            $restoreArgs = @('spo', 'site', 'recyclebinitem', 'restore', '--siteUrl', $SiteUrl.AbsoluteUri)
            if ($AllPrimary) {
                $restoreArgs += '--allPrimary'
            }
            if ($AllSecondary) {
                $restoreArgs += '--allSecondary'
            }

            $restoreOutput = m365 @restoreArgs 2>&1
            if ($LASTEXITCODE -eq 0) {
                $script:Summary.OperationStatus = 'Completed'
                $script:Summary.Results += [pscustomobject]@{
                    Scope   = $scopeLabel
                    Status  = 'Restored'
                    Message = 'Restore request submitted successfully.'
                }
            }
            else {
                $script:Summary.OperationStatus = 'Failed'
                $failureMessage = if ([string]::IsNullOrWhiteSpace($restoreOutput)) {
                    'Restore failed. See CLI output for details.'
                }
                else {
                    $restoreOutput.ToString()
                }

                $script:Summary.Failures += [pscustomobject]@{
                    Stage   = 'ScopeRestore'
                    Target  = $scopeLabel
                    Message = $failureMessage
                }

                $script:Summary.Results += [pscustomobject]@{
                    Scope   = $scopeLabel
                    Status  = 'Failed'
                    Message = $failureMessage
                }

                Write-Warning "Restore command failed for scope '$scopeLabel'. CLI output: $failureMessage"
            }
        }
    }

    end {
        $outputPath = $script:OutputCsvPath
        if ($script:ParameterSetName -eq 'Ids') {
            $results = $script:Summary.Results
            if ($outputPath -and $results.Count -gt 0) {
                Write-Verbose "Exporting results to '$outputPath'."
                $destinationDir = Split-Path -Parent $outputPath
                if ($destinationDir -and -not (Test-Path -Path $destinationDir)) {
                    New-Item -ItemType Directory -Path $destinationDir -Force > $null
                }
                $results | Export-Csv -Path $outputPath -NoTypeInformation
            }
        }

        Write-Host '--- Restore summary ---'
        Write-Host "Site URL          : $($script:Summary.SiteUrl)"
        Write-Host "Mode              : $($script:Summary.Mode)"
        Write-Host "Scope             : $($script:Summary.Scope)"
        Write-Host "Status            : $($script:Summary.OperationStatus)"

        if ($script:Summary.TotalItems -ne $null) {
            Write-Host "Items requested   : $($script:Summary.TotalItems)"
        }
        if ($script:Summary.BatchSize -ne $null) {
            Write-Host "Batch size        : $($script:Summary.BatchSize)"
        }
        if ($script:Summary.BatchesAttempted -ne $null) {
            Write-Host "Batches attempted : $($script:Summary.BatchesAttempted)"
        }
        if ($script:Summary.Restored -ne $null) {
            Write-Host "Items restored    : $($script:Summary.Restored)"
        }
        if ($script:Summary.Skipped -ne $null) {
            Write-Host "Items skipped     : $($script:Summary.Skipped)"
        }
        if ($script:Summary.Failed -ne $null) {
            Write-Host "Items failed      : $($script:Summary.Failed)"
        }
        if ($outputPath) {
            Write-Host "Output CSV        : $outputPath"
        }

        [pscustomobject]$script:Summary
    }
}

# Example usage
# Restore individual recycle bin items from a CSV export
# Restore-SpoRecycleBinItems -SiteUrl 'https://contso.sharepoint.com/sites/RestoreDocs' -InputCsvPath './recyclebin.csv' -OutputCsvPath './recyclebin-results.csv' -BatchSize 25 -Verbose

# Restore all items from the first-stage recycle bin
# Restore-SpoRecycleBinItems -SiteUrl 'https://contso.sharepoint.com/sites/RestoreDocs' -AllPrimary -Verbose
```

# [PnP PowerShell](#tab/pnpps)

```powershell
# Input file
$Path = "$PSScriptRoot\recyclebin.csv"
# Output file
$OutputFile = "$PSScriptRoot\recyclebinresults.csv"

$NoInBatch = 10

$ErrorActionPreference = 'Stop'
$InformationPreference = 'Continue'
Connect-PnPOnline -url:"https://contso.sharepoint.com/sites/RestoreDocs" -PnPManagementShell

function Start-Processing {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [string]
        $csvFilePath,
        [Parameter(Mandatory = $true)]
        [int]
        $processBatchCount
    )

    $csvItems = Get-Content -path:$csvFilePath | ConvertFrom-csv
    $recycleBinSplit = Split-Array -InputObject $csvItems -Size $processBatchCount

    $batchCount = $recycleBinSplit.Count
    $i = 0
    if($recycleBinSplit.Count -eq $csvItems.Count)
    {
        Write-Information -MessageData:"Restoring deleted items batch 1 of 1 containing $($recycleBinSplit.Count) items..."
        Restore-RecycleBinItems -Ids:$recycleBinSplit
    }
    else {
        $recycleBinSplit | ForEach-Object {
            $items = $PSItem
            $i++;
            Write-Information -MessageData:"Restoring deleted items batch $i of $batchCount containing $($items.Count)..."
            Restore-RecycleBinItems -Ids:$items
        }
    }
}

function Split-Array {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory)]
        [object[]] $InputObject,
        [int] $Size = 10
    )
    $outArray = @()
    $parts = [math]::Ceiling($InputObject.Count / $Size)

    for ($i = 0; $i -le $parts - 1; $i++) {
        $start = $i * $Size
        $end = (($i + 1) * $Size) - 1
        $outArray += , @($InputObject[$start..$end])
    }

    Write-Output $outArray
}

function Restore-RecycleBinItems {
    param(
        [Parameter(Mandatory)]
        [Object[]]
        $Ids
    )


    $apiCall = "/_api/site/RecycleBin/RestoreByIds"
    $idsString = ($Ids).Id -join "','"
    $body = "{'ids':['$idsString']}"

    try {
        Invoke-PnPSPRestMethod -Method Post -Url $apiCall -Content $body | Out-Null
        Write-Information "Batch Success"
        $Ids | ForEach-Object {
            $id = $PSItem
            $id | Add-Member -MemberType NoteProperty -Name "Status" -Value "Success"
            Write-Output $id
        }
    }
    catch {
        $Exception = $_
        Write-Warning "Unable to process as batch, processing individually...."
        $Ids | ForEach-Object {
            $id = $PSItem
            try {
                $body = "{'ids':['$($id.Id)']}"
                Invoke-PnPSPRestMethod -Method Post -Url $apiCall -Content $body | Out-Null
                Write-Information "Success: $($id.Id)"
                $id | Add-Member -MemberType NoteProperty -Name "Status" -Value "Success"
                Write-Output $id
            }
            catch {
                $Exception = $_
                $odataError = $Exception.Exception.Message | ConvertFrom-Json
                $message = $odataError.'odata.error'.message.value
                if ($message.Contains("Value does not fall within the expected range.") -eq $true) {
                    $message = "No longer in recycle bin / Previously restored"
                }

                $id | Add-Member -MemberType NoteProperty -Name "Status" -Value $message
                Write-Information "Failed: $($id.Id) - $message"
                Write-Output $id
            }
        }
    }
}

Write-Information -MessageData:"Processing file $Path and restoring recycle bin items in batches of $NoInBatch..."
Start-Processing -csvFilePath:$Path -processBatchCount:$NoInBatch | Export-Csv $OutputFile -NoTypeInformation
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s)                                      |
| ---------------------------------------------- |
| [Paul Matthews](https://github.com/pmatthews05) |
| [Adam Wójcik](https://github.com/Adam-it)       |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/bulk-restore-from-recycle-bin" aria-hidden="true" />

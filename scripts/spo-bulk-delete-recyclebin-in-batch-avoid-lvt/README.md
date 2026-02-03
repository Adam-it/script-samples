

# Bulk delete recycle bin items from a site in batch

## Summary

This script allows you to bulk delete recycle bin items from a SharePoint Online site while avoiding List View Threshold issues by using batch processing. The CLI for Microsoft 365 version provides a modern, streamlined approach with comprehensive error handling and audit capabilities.

> [!IMPORTANT]
> This script permanently deletes items from the recycle bin. Items cannot be restored once deleted. Always test with `-WhatIf` parameter first.

[!INCLUDE [Delete Warning](../../docfx/includes/DELETE-WARN.md)]

![Example Screenshot](assets/example.png)

### Prerequisites

- The user account that runs the script must have permissions to manage the recycle bin on the SharePoint Online site.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Email of user who deleted items (filter)")]
    [ValidatePattern('^[^@]+@[^@]+\\.[^@]+$')]
    [string]$DeletedByEmail,

    [Parameter(Mandatory = $false, HelpMessage = "Start date for deletion filter (default: 8 days ago)")]
    [DateTime]$DateFrom = (Get-Date).AddDays(-8),

    [Parameter(Mandatory = $false, HelpMessage = "End date for deletion filter (default: 5 days ago)")]
    [DateTime]$DateTo = (Get-Date).AddDays(-5),

    [Parameter(Mandatory = $false, HelpMessage = "Number of items to delete per batch (default: 10)")]
    [ValidateRange(1, 100)]
    [int]$BatchSize = 10,

    [Parameter(Mandatory = $false, HelpMessage = "Path for CSV exports (default: current directory)")]
    [string]$ExportPath = (Get-Location).Path,

    [Parameter(Mandatory = $false, HelpMessage = "Target secondary recycle bin")]
    [switch]$Secondary
)

begin {
    Write-Verbose "Authenticating to Microsoft 365..."
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to Microsoft 365"
    }

    if ($PSBoundParameters.ContainsKey('ExportPath')) {
        if (-not (Test-Path -Path $ExportPath -PathType Container)) {
            throw "Export path does not exist: $ExportPath"
        }
    }

    if ($DateFrom -ge $DateTo) {
        throw "DateFrom must be earlier than DateTo"
    }

    $script:Summary = @{
        TotalItems      = 0
        FilteredItems   = 0
        BatchesAttempted = 0
        ItemsDeleted    = 0
        Failures        = 0
    }

    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $script:ItemsExportPath = Join-Path $ExportPath "RecycleBinItems_$timestamp.csv"
    $script:ResultsExportPath = Join-Path $ExportPath "RecycleBinResults_$timestamp.csv"
    $script:ReportCollection = [System.Collections.Generic.List[object]]::new()

    Start-Transcript -Path (Join-Path $ExportPath "RecycleBinDeletion_$timestamp.log")
}

process {
    Write-Host "`nRetrieving recycle bin items from site: $SiteUrl" -ForegroundColor Cyan

    try {
        $commandArgs = @('spo', 'site', 'recyclebinitem', 'list', '--siteUrl', $SiteUrl, '--output', 'json')
        if ($Secondary) {
            $commandArgs += '--secondary'
        }

        $result = m365 @commandArgs 2>&1 | Out-String
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve recycle bin items. Exit code: $LASTEXITCODE"
        }

        $recycleBinItems = @($result | ConvertFrom-Json)
        $script:Summary.TotalItems = $recycleBinItems.Count

        Write-Host "Total items in recycle bin: $($recycleBinItems.Count)" -ForegroundColor White

        $filteredItems = $recycleBinItems | Where-Object {
            $matchesEmail = $true
            $matchesDate = $true

            if ($PSBoundParameters.ContainsKey('DeletedByEmail')) {
                $matchesEmail = $_.DeletedByEmail -eq $DeletedByEmail
            }

            if ($_.DeletedDate) {
                $deletedDate = [DateTime]::Parse($_.DeletedDate)
                $matchesDate = ($deletedDate -ge $DateFrom) -and ($deletedDate -le $DateTo)
            }

            $matchesEmail -and $matchesDate
        }

        $script:Summary.FilteredItems = $filteredItems.Count

        if ($filteredItems.Count -eq 0) {
            Write-Host "No items found matching the filter criteria." -ForegroundColor Yellow
            return
        }

        Write-Host "Items matching filter criteria: $($filteredItems.Count)" -ForegroundColor Green
        $filteredItems | Export-Csv -Path $script:ItemsExportPath -NoTypeInformation
        Write-Verbose "Exported filtered items to: $script:ItemsExportPath"

        $batches = for ($i = 0; $i -lt $filteredItems.Count; $i += $BatchSize) {
            $end = [Math]::Min($i + $BatchSize - 1, $filteredItems.Count - 1)
            , $filteredItems[$i..$end]
        }

        Write-Host "`nDeleting $($filteredItems.Count) items in $($batches.Count) batch(es) of up to $BatchSize items..." -ForegroundColor Cyan

        foreach ($batch in $batches) {
            $script:Summary.BatchesAttempted++
            $batchNumber = $script:Summary.BatchesAttempted
            $batchItemCount = $batch.Count

            Write-Verbose "Processing batch $batchNumber of $($batches.Count) ($batchItemCount items)"

            try {
                if ($PSCmdlet.ShouldProcess("Batch $batchNumber ($batchItemCount items)", 'Delete from recycle bin')) {
                    $ids = ($batch | ForEach-Object { $_.Id }) -join ','

                    m365 spo site recyclebinitem remove --siteUrl $SiteUrl --ids $ids --force 2>&1 | Out-Null

                    if ($LASTEXITCODE -ne 0) {
                        throw "CLI command failed with exit code $LASTEXITCODE"
                    }

                    Write-Host "  Batch $batchNumber: Successfully deleted $batchItemCount items" -ForegroundColor Green
                    $script:Summary.ItemsDeleted += $batchItemCount

                    foreach ($item in $batch) {
                        $script:ReportCollection.Add([PSCustomObject]@{
                            Id               = $item.Id
                            Title            = $item.Title
                            DirName          = $item.DirName
                            LeafName         = $item.LeafName
                            DeletedByEmail   = $item.DeletedByEmail
                            DeletedDate      = $item.DeletedDate
                            BatchNumber      = $batchNumber
                            Status           = 'Success'
                            ErrorMessage     = ''
                        })
                    }
                } else {
                    Write-Host "  Batch $batchNumber: WhatIf - Would delete $batchItemCount items" -ForegroundColor Yellow
                    $script:Summary.ItemsDeleted += $batchItemCount
                }
            }
            catch {
                Write-Warning "Batch $batchNumber failed: $($_.Exception.Message)"
                $script:Summary.Failures += $batchItemCount

                foreach ($item in $batch) {
                    $script:ReportCollection.Add([PSCustomObject]@{
                        Id               = $item.Id
                        Title            = $item.Title
                        DirName          = $item.DirName
                        LeafName         = $item.LeafName
                        DeletedByEmail   = $item.DeletedByEmail
                        DeletedDate      = $item.DeletedDate
                        BatchNumber      = $batchNumber
                        Status           = 'Failed'
                        ErrorMessage     = $_.Exception.Message
                    })
                }

                continue
            }
        }
    }
    catch {
        Write-Warning "Failed to process recycle bin items: $($_.Exception.Message)"
        throw
    }
}

end {
    Stop-Transcript

    if ($script:ReportCollection.Count -gt 0) {
        $script:ReportCollection | Export-Csv -Path $script:ResultsExportPath -NoTypeInformation
        Write-Host "`nResults exported to: $script:ResultsExportPath" -ForegroundColor Cyan
    }

    Write-Host "`n===== Summary =====" -ForegroundColor Cyan
    Write-Host "Site: $SiteUrl" -ForegroundColor White
    Write-Host "Date range: $($DateFrom.ToString('yyyy-MM-dd')) to $($DateTo.ToString('yyyy-MM-dd'))" -ForegroundColor White
    if ($PSBoundParameters.ContainsKey('DeletedByEmail')) {
        Write-Host "Deleted by: $DeletedByEmail" -ForegroundColor White
    }
    Write-Host "Total items in recycle bin: $($script:Summary.TotalItems)" -ForegroundColor White
    Write-Host "Items matching filter: $($script:Summary.FilteredItems)" -ForegroundColor White
    Write-Host "Batches attempted: $($script:Summary.BatchesAttempted)" -ForegroundColor White
    Write-Host "Items deleted: $($script:Summary.ItemsDeleted)" -ForegroundColor Green

    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor White
    }
}

# Example 1: Delete items deleted by specific user in date range
# .\Remove-RecycleBinItemsBulk.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -DeletedByEmail "user@contoso.com" -DateFrom (Get-Date).AddDays(-10) -DateTo (Get-Date).AddDays(-3)

# Example 2: Test with WhatIf (no deletions performed)
# .\Remove-RecycleBinItemsBulk.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -DeletedByEmail "user@contoso.com" -WhatIf

# Example 3: Delete with custom batch size and verbose output
# .\Remove-RecycleBinItemsBulk.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -BatchSize 20 -Verbose

# Example 4: Delete from secondary recycle bin
# .\Remove-RecycleBinItemsBulk.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/sales" -DeletedByEmail "user@contoso.com" -Secondary

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell

######################################################### 
# Step 1: Execute this part only and wait for it to complete.
# This step will get the items from the recycle bin based on the defined condition in CSV
######################################################### 

## PnP PowerShell 

$today = (Get-Date)  

# Specify date from  
$date1 = $today.Date.AddDays(-8)  

# Specify date to  
$date2 = $today.Date.AddDays(-5)  

Connect-PnPOnline -Url https://tenant.sharepoint.com/sites/repro -Interactive 

$recycleBinItems = Get-PnPRecycleBinItem -RowLimit 999999 | ? { 
    ($_.DeletedByEmail -eq 'first.last@tenant.onmicrosoft.com') -and 
    (($_.DeletedDate -gt $date1) -and ($_.DeletedDate -lt $Date2))
}

$recycleBinItems | Export-Csv C:\recyclebin.csv -NoTypeInformation 

# Open CSV and remove rows you do not wish to delete. Save the CSV file.

######################################################### 
# Step 2: Now execute the below part and wait for it to complete.
# This step will fetch the items from the CSV report stored locally and delete the items by IDs in a batch of 10 items (default)
######################################################### 

# Input file 
$Path = "C:\recyclebin.csv" 

# Output file 
$OutputFile = "C:\recyclebinresults.csv" 

$NoInBatch = 10 

$ErrorActionPreference = 'Stop' 
$InformationPreference = 'Continue' 

Connect-PnPOnline -Url "https://tenant.sharepoint.com/sites/repro" -Interactive 

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

    $csvItems = Get-Content -Path $csvFilePath | ConvertFrom-Csv 
    $recycleBinSplit = Split-Array -InputObject $csvItems -Size $processBatchCount 

    $batchCount = $recycleBinSplit.Count 
    $i = 0 

    if ($recycleBinSplit.Count -eq $csvItems.Count) { 
        Write-Information -MessageData "Purging deleted items batch 1 of 1 containing $($recycleBinSplit.Count) items..." 
        Clear-RecycleBinItems -Ids $recycleBinSplit 
    } else { 
        $recycleBinSplit | ForEach-Object { 
            $items = $PSItem 
            $i++
            Write-Information -MessageData "Purging deleted items batch $i of $batchCount containing $($items.Count)..." 
            Clear-RecycleBinItems -Ids $items 
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

function Clear-RecycleBinItems { 
    param( 
        [Parameter(Mandatory)] 
        [Object[]] 
        $Ids 
    ) 

    $apiCall = "/_api/site/RecycleBin/DeleteByIds" 
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
    } catch { 
        $Exception = $_ 
        Write-Warning "Unable to process as a batch, processing individually...." 
        $Ids | ForEach-Object { 
            $id = $PSItem 
            try { 
                $body = "{'ids':['$($id.Id)']}" 
                Invoke-PnPSPRestMethod -Method Post -Url $apiCall -Content $body | Out-Null 
                Write-Information "Success: $($id.Id)" 
                $id | Add-Member -MemberType NoteProperty -Name "Status" -Value "Success" 
                Write-Output $id 
            } catch { 
                $Exception = $_ 
                $odataError = $Exception.Exception.Message | ConvertFrom-Json 
                $message = $odataError.'odata.error'.message.value 

                if ($message.Contains("Value does not fall within the expected range.") -eq $true) { 
                    $message = "No longer in recycle bin / Previously deleted" 
                } 

                $id | Add-Member -MemberType NoteProperty -Name "Status" -Value $message 
                Write-Information "Failed: $($id.Id) - $message" 
                Write-Output $id 
            } 
        } 
    } 
} 

Write-Information -MessageData "Processing file $Path and purging recycle bin items in batches of $NoInBatch..."

Start-Processing -csvFilePath $Path -processBatchCount $NoInBatch | Export-Csv $OutputFile -NoTypeInformation

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Restore large amount of items from SharePoint Recycle bin in bulk](https://pnp.github.io/script-samples/bulk-restore-from-recyclebin/README.html)

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Eilaf Barmare |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-bulk-delete-recyclebin-in-batch-avoid-lvt" aria-hidden="true" />

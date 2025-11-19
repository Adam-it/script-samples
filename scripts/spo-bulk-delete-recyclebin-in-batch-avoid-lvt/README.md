

# Bulk delete recycle bin items from a site in batch

## Summary

This script performs a two-step process to manage the recycle bin items in a SharePoint Online site. Here's a summary of the script's functionality: 

Step 1: 
- The script connects to a SharePoint Online site using PnP PowerShell. 
- It defines a date range and retrieves recycle bin items meeting specific conditions based on a defined CSV file. 
- The retrieved items are exported to a CSV file named "recyclebin.csv". 

Step 2: 
- The script connects again to the same SharePoint Online site using PnP PowerShell. 
- It reads the "recyclebin.csv" file, which should have been manually modified to contain only the items intended for deletion. 
- The script processes the items in batches (default batch size: 10) and deletes them from the recycle bin using SharePoint's REST API. 
- The results of the deletion process are written to "recyclebinresults.csv". 

Both steps include informative messages to keep users updated on the progress and status of the operations. 

> [!Note]
> The script relies on the PnP PowerShell module to interact with SharePoint Online, and it is essential to have the module installed and authenticated before executing the script. Additionally, users should carefully review and modify the "recyclebin.csv" file in Step 2 to ensure that only the intended items are deleted. 

[!INCLUDE [Delete Warning](../../docfx/includes/DELETE-WARN.md)]

![Example Screenshot](assets/example.png)

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

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Remove-SpoRecycleBinItemsWithCli {
    <#
    .SYNOPSIS
    Deletes SharePoint Online recycle bin items in batches using CLI for Microsoft 365.

    .DESCRIPTION
    Retrieves recycle bin items matching the CSV filter, exports them for review, and deletes selected
    entries in configurable batches, avoiding List View Threshold issues.

    .PARAMETER SiteUrl
    URL of the SharePoint Online site whose recycle bin should be queried.

    .PARAMETER FilterPath
    Path to the CSV file describing the initial filter criteria (DeletedByEmail, DeletedDateFrom, DeletedDateTo).

    .PARAMETER ItemsCsvPath
    Output CSV containing candidate recycle bin items (defaults to ./recyclebin.csv).

    .PARAMETER ResultsCsvPath
    Output CSV capturing deletion results (defaults to ./recyclebinresults.csv).

    .PARAMETER BatchSize
    Number of items to delete per batch (default 10).

    .PARAMETER Force
    Suppress confirmation prompts when deleting batches.

    .EXAMPLE
    Remove-SpoRecycleBinItemsWithCli -SiteUrl https://contoso.sharepoint.com/sites/repro `
        -FilterPath ./filter.csv -BatchSize 20 -Force
    #>
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter(Mandatory, HelpMessage = 'URL of the SharePoint Online site recycle bin.')]
        [ValidateNotNullOrEmpty()]
        [string]$SiteUrl,

        [Parameter(Mandatory, HelpMessage = 'CSV file describing filter criteria.')]
        [ValidateNotNullOrEmpty()]
        [string]$FilterPath,

        [Parameter()]
        [ValidateNotNullOrEmpty()]
        [string]$ItemsCsvPath = './recyclebin.csv',

        [Parameter()]
        [ValidateNotNullOrEmpty()]
        [string]$ResultsCsvPath = './recyclebinresults.csv',

        [Parameter()]
        [ValidateRange(1,100)]
        [int]$BatchSize = 10,

        [Parameter()]
        [switch]$Force
    )

    begin {
        if (-not (Test-Path -Path $FilterPath -PathType Leaf)) {
            throw "Filter file '$FilterPath' was not found."
        }

        Write-Host 'Ensuring CLI authentication.' -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "CLI login failed. Output: $loginOutput"
        }

        try {
            $script:Filter = Import-Csv -Path $FilterPath -ErrorAction Stop
        }
        catch {
            throw "Unable to parse filter CSV. $($_.Exception.Message)"
        }

        if ($script:Filter.Count -eq 0) {
            throw 'Filter CSV is empty. Provide at least one row with criteria.'
        }

        $script:Items = @()
        $script:Results = @()
    }

    process {
        Write-Host 'Retrieving recycle bin items.' -ForegroundColor Cyan
        $itemsJson = m365 spo site recyclebinitem list --siteUrl $SiteUrl --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Unable to list recycle bin items. CLI output: $itemsJson"
        }

        try {
            $allItems = $itemsJson | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse recycle bin list. $($_.Exception.Message)"
        }

        if (-not $allItems) {
            Write-Warning 'Recycle bin is empty.'
            return
        }

        foreach ($row in $script:Filter) {
            $deletedBy = $row.DeletedByEmail
            $fromDate = if ($row.DeletedDateFrom) { Get-Date $row.DeletedDateFrom } else { Get-Date '1900-01-01' }
            $toDate = if ($row.DeletedDateTo) { Get-Date $row.DeletedDateTo } else { Get-Date }

            $matched = $allItems | Where-Object {
                ($deletedBy -and $_.DeletedByEmail -eq $deletedBy) -and
                ([datetime]$_.DeletedDate -ge $fromDate) -and
                ([datetime]$_.DeletedDate -le $toDate)
            }

            if ($matched) {
                $script:Items += $matched
            }
        }

        $script:Items = $script:Items | Sort-Object -Property Id -Unique

        if ($script:Items.Count -eq 0) {
            Write-Warning 'No items matched the filter. Review your CSV criteria.'
            return
        }

        Write-Host "Exporting candidate items to '$ItemsCsvPath'." -ForegroundColor Cyan
        $script:Items | Export-Csv -Path $ItemsCsvPath -Encoding UTF8 -NoTypeInformation

        Write-Host "Review '$ItemsCsvPath' and remove rows you do not want to delete." -ForegroundColor Yellow

        if ($Force.IsPresent) {
            Write-Host 'Batch deletion triggered with -Force.' -ForegroundColor Cyan
            $itemsToDelete = @((Import-Csv -Path $ItemsCsvPath))

            if ($itemsToDelete.Count -eq 0) {
                Write-Warning 'No items remained in the review CSV. Aborting deletion.'
                return
            }

            $total = $itemsToDelete.Count
            for ($offset = 0; $offset -lt $total; $offset += $BatchSize) {
                $endIndex = [Math]::Min($offset + $BatchSize - 1, $total - 1)
                $batch = $itemsToDelete[$offset..$endIndex]
                $batchNumber = [int]($offset / $BatchSize) + 1
                $ids = ($batch | ForEach-Object { $_.Id }) -join ','
                $action = "Delete recycle bin items batch $batchNumber containing $($batch.Count) entries"
                if (-not ($Force.IsPresent -or $PSCmdlet.ShouldProcess($SiteUrl, $action))) {
                    continue
                }

                Write-Host $action -ForegroundColor Cyan
                $deleteOutput = m365 spo site recyclebinitem remove --siteUrl $SiteUrl --ids $ids --force 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Batch $batchNumber failed. CLI output: $deleteOutput"
                    foreach ($entry in $batch) {
                        $script:Results += [pscustomobject]@{
                            Id     = $entry.Id
                            Status = 'Failed'
                            Notes  = $deleteOutput
                        }
                    }
                    continue
                }

                foreach ($entry in $batch) {
                    $script:Results += [pscustomobject]@{
                        Id     = $entry.Id
                        Status = 'Success'
                        Notes  = ''
                    }
                }
            }
        }
    }

    end {
        if ($script:Results.Count -gt 0) {
            $script:Results | Export-Csv -Path $ResultsCsvPath -Encoding UTF8 -NoTypeInformation
            Write-Host "Deletion results saved to '$ResultsCsvPath'." -ForegroundColor Cyan
        }

        Write-Host "Items exported: $(@($script:Items).Count)" -ForegroundColor Cyan
        Write-Host "Items deleted: $(@($script:Results | Where-Object { $_.Status -eq 'Success' }).Count)" -ForegroundColor Green
        Write-Host "Items failed: $(@($script:Results | Where-Object { $_.Status -eq 'Failed' }).Count)" -ForegroundColor Red
    }
}

# example usage
Remove-SpoRecycleBinItemsWithCli -SiteUrl https://contoso.sharepoint.com/sites/repro -FilterPath ./filter.csv -Force
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

Sample first appeared on [Restore large amount of items from SharePoint Recycle bin in bulk](https://pnp.github.io/script-samples/bulk-restore-from-recyclebin/README.html)

## Contributors

| Author(s) |
|-----------|
| Eilaf Barmare |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-bulk-delete-recyclebin-in-batch-avoid-lvt" aria-hidden="true" />

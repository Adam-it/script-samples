
# Restore multiple deleted items from the SharePoint Online Recycle Bin based on deletion date and user name

## Summary

This PowerShell script is designed to help SharePoint Online administrators to restore items from the recycle bin that were deleted by a specific account (such as "System Account" or a SharePoint App) within a user-defined number of days.

## Scenario

Sometimes, there's a need to restore files that were accidentally deleted by users. One common scenario is when a user deletes a synced SharePoint folder without properly disconnecting it first.

## Requirements

To run this PowerShell script successfully, ensure the following:

1. PowerShell Version: [PowerShell 7 or later](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.5)
2. PnP PowerShell Module: Installed and imported [(Install-Module PnP.PowerShell)](https://pnp.github.io/powershell/articles/installation.html)
3. [App-Only Authentication](https://github.com/pnp/PnP-PowerShell/tree/master/Samples/SharePoint.ConnectUsingAppPermissions): You must have:
    + A registered Azure AD App with appropriate SharePoint permissions
    + A valid Client ID
    + A certificate installed locally with its thumbprint
4. SharePoint Online Access: The app must have access to the target SharePoint site
5. Log Directory: The specified log directory must exist on the local machine

[More about Restore-PnPRecycleBinItem](https://pnp.github.io/powershell/cmdlets/Restore-PnPRecycleBinItem.html)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Name of user who deleted items")]
    [string]$DeletedByName,

    [Parameter(Mandatory = $true, HelpMessage = "Number of days to look back")]
    [ValidateRange(1, 93)]
    [int]$DaysBack,

    [Parameter(Mandatory = $false, HelpMessage = "Restore from second-stage recycle bin")]
    [switch]$Secondary,

    [Parameter(Mandatory = $false, HelpMessage = "Output folder for CSV and transcript")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath)) {
            throw "Output path does not exist: $OutputPath"
        }
    }

    Write-Host "Logging in to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to Microsoft 365"
    }
    Write-Host "Successfully logged in" -ForegroundColor Green

    $script:Summary = @{
        ItemsFound = 0
        ItemsRestored = 0
        Failures = 0
    }

    $script:ReportCollection = @()

    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path -Path $OutputPath -ChildPath "RestoreItems_$timestamp.log"
    Start-Transcript -Path $transcriptPath | Out-Null
    Write-Host "Transcript started: $transcriptPath" -ForegroundColor White

    $script:cutoffDate = (Get-Date).AddDays(-$DaysBack)
    Write-Host "Searching for items deleted by '$DeletedByName' since $($script:cutoffDate.ToString('yyyy-MM-dd'))" -ForegroundColor White
}

process {
    try {
        Write-Verbose "Retrieving recycle bin items from $SiteUrl"
        
        $listArgs = @('spo', 'site', 'recyclebinitem', 'list', '--siteUrl', $SiteUrl, '--output', 'json')
        if ($Secondary) {
            $listArgs += '--secondary'
            Write-Host "Searching in second-stage recycle bin" -ForegroundColor White
        }

        $itemsJson = m365 @listArgs 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve recycle bin items: $itemsJson"
        }

        $allItems = @($itemsJson | ConvertFrom-Json)

        $items = $allItems | Where-Object {
            $_.DeletedByName -eq $DeletedByName -and
            [datetime]$_.DeletedDate -gt $script:cutoffDate
        }

        if ($items.Count -eq 0) {
            Write-Warning "No items found matching criteria"
            return
        }

        Write-Host "`nFound $($items.Count) items to restore:" -ForegroundColor Cyan
        $script:Summary.ItemsFound = $items.Count

        foreach ($item in $items) {
            Write-Host "  - $($item.Title) (Deleted: $($item.DeletedDate))" -ForegroundColor White
        }

        $target = "$($items.Count) items from recycle bin"
        $action = "Restore to original location"

        if ($PSCmdlet.ShouldProcess($target, $action)) {
            try {
                Write-Host "`nRestoring items..." -ForegroundColor Cyan
                $ids = ($items.Id -join ',')

                # Try batch restore first (fast path)
                m365 spo site recyclebinitem restore --siteUrl $SiteUrl --ids $ids 2>&1 | Out-Null

                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Batch restore failed. Falling back to individual item restore..."
                    
                    # Fallback: Restore items individually (resilient path)
                    $currentItem = 0
                    foreach ($item in $items) {
                        $currentItem++
                        Write-Progress -Activity "Restoring Items" -Status "Processing $currentItem of $($items.Count): $($item.Title)" -PercentComplete (($currentItem / $items.Count) * 100)
                        
                        try {
                            m365 spo site recyclebinitem restore --siteUrl $SiteUrl --ids $item.Id 2>&1 | Out-Null
                            
                            if ($LASTEXITCODE -eq 0) {
                                Write-Verbose "Restored: $($item.Title)"
                                $script:Summary.ItemsRestored++
                                
                                $script:ReportCollection += [PSCustomObject]@{
                                    Title = $item.Title
                                    Id = $item.Id
                                    DeletedByName = $item.DeletedByName
                                    DeletedDate = $item.DeletedDate
                                    DirName = $item.DirName
                                    ItemType = $item.ItemType
                                    Size = $item.Size
                                    Status = "Restored (Individual)"
                                    ErrorMessage = ""
                                }
                            } else {
                                throw "CLI command failed"
                            }
                        }
                        catch {
                            Write-Warning "Failed to restore '$($item.Title)': $($_.Exception.Message)"
                            $script:Summary.Failures++
                            
                            $script:ReportCollection += [PSCustomObject]@{
                                Title = $item.Title
                                Id = $item.Id
                                DeletedByName = $item.DeletedByName
                                DeletedDate = $item.DeletedDate
                                DirName = $item.DirName
                                ItemType = $item.ItemType
                                Size = $item.Size
                                Status = "Failed"
                                ErrorMessage = $_.Exception.Message
                            }
                        }
                    }
                    
                    Write-Progress -Activity "Restoring Items" -Completed
                } else {
                    # Batch restore succeeded
                    Write-Host "SUCCESS: Restored $($items.Count) items (batch mode)" -ForegroundColor Green
                    $script:Summary.ItemsRestored = $items.Count
                    
                    foreach ($item in $items) {
                        $script:ReportCollection += [PSCustomObject]@{
                            Title = $item.Title
                            Id = $item.Id
                            DeletedByName = $item.DeletedByName
                            DeletedDate = $item.DeletedDate
                            DirName = $item.DirName
                            ItemType = $item.ItemType
                            Size = $item.Size
                            Status = "Restored (Batch)"
                            ErrorMessage = ""
                        }
                    }
                }
                    }
                }
            }
            catch {
                Write-Warning "Unexpected error during restore operation: $($_.Exception.Message)"
                $script:Summary.Failures++
                foreach ($item in $items) {
                    $script:ReportCollection += [PSCustomObject]@{
                        Title = $item.Title
                        Id = $item.Id
                        DeletedByName = $item.DeletedByName
                        DeletedDate = $item.DeletedDate
                        DirName = $item.DirName
                        ItemType = $item.ItemType
                        Size = $item.Size
                        Status = "Failed"
                        ErrorMessage = $_.Exception.Message
                    }
                }
            }
        }
        else {
            Write-Host "`nWHATIF: Would restore $($items.Count) items" -ForegroundColor Yellow
            $script:Summary.ItemsRestored = $items.Count

            foreach ($item in $items) {
                $script:ReportCollection += [PSCustomObject]@{
                    Title = $item.Title
                    Id = $item.Id
                    DeletedByName = $item.DeletedByName
                    DeletedDate = $item.DeletedDate
                    DirName = $item.DirName
                    ItemType = $item.ItemType
                    Size = $item.Size
                    Status = "WhatIf"
                    ErrorMessage = ""
                }
            }
        }
    }
    catch {
        Write-Error "Error during restoration: $($_.Exception.Message)"
        $script:Summary.Failures++
    }
}

end {
    Stop-Transcript | Out-Null

    if ($script:ReportCollection.Count -gt 0) {
        $csvPath = Join-Path -Path $OutputPath -ChildPath "RestoredItems_$timestamp.csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation
        Write-Host "`nCSV report saved: $csvPath" -ForegroundColor White
    }

    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Items Found    : $($script:Summary.ItemsFound)" -ForegroundColor White
    Write-Host "Items Restored : $($script:Summary.ItemsRestored)" -ForegroundColor Green
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures       : $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures       : 0" -ForegroundColor White
    }
    Write-Host "============================" -ForegroundColor Cyan
}

# Restore items deleted by "System Account" in last 7 days with WhatIf
# .\Restore-MultipleItems.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -DeletedByName "System Account" -DaysBack 7 -WhatIf

# Restore items deleted by specific user in last 30 days
# .\Restore-MultipleItems.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -DeletedByName "john@contoso.com" -DaysBack 30

# Restore from second-stage recycle bin
# .\Restore-MultipleItems.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -DeletedByName "System Account" -DaysBack 14 -Secondary

# Restore with verbose output
# .\Restore-MultipleItems.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -DeletedByName "System Account" -DaysBack 7 -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

function Get-UserInput {
    [CmdletBinding()]
    param ()

    try {
        $inputData = [ordered]@{}
        $inputData["siteURL"] = Read-Host "Enter the SharePoint site URL (e.g., https://yourtenant.sharepoint.com/sites/demo)"
        $inputData["tenant"] = Read-Host "Enter your tenant domain (e.g., yourtenant.onmicrosoft.com)"
        $inputData["clientID"] = Read-Host "Enter the Azure AD App Client ID"
        $inputData["thumbprint"] = Read-Host "Enter the certificate thumbprint"
        $inputData["deletedByName"] = Read-Host "Enter the name of the account that deleted the items (e.g., System Account)"
        $inputData["logLocation"] = Read-Host "Enter the full path to the log directory (e.g., C:\Temp\Logs)"
        $inputData["numberOfDays"] = Read-Host "Enter the number of days to look back for deleted items"     

        return $inputData
    } catch {
        Write-Error "Error collecting user input: $_"
        exit 1
    }
}

function Test-LogDirectoryPath {
    param ([string]$Path)
    try {
        if (-not (Test-Path -Path $Path)) {
            throw "Log directory does not exist: $Path"
        }
        Write-Host "Log directory exists: $Path"
    } catch {
        Write-Error "Log directory validation failed: $_"
        exit 1
    }
}

function Connect-ToSharePoint {
    param (
        [string]$Tenant,
        [string]$ClientID,
        [string]$Thumbprint,
        [string]$SiteURL
    )
    try {
        Connect-PnPOnline -Tenant $Tenant -ClientId $ClientID -Thumbprint $Thumbprint -Url $SiteURL
        Write-Host "Connected to SharePoint site: $SiteURL"
    } catch {
        Write-Error "Failed to connect to SharePoint: $_"
        exit 1
    }
}

function Restore-RecycleBinItems {
    param (
        [string]$DeletedByName,
        [datetime]$TargetDate,
        [int]$BatchSize,
        [string]$LogFile
    )

    try {
        $count = 0
        Write-Host "Retrieving items deleted by '$DeletedByName' since $TargetDate..."
        $items = Get-PnPRecycleBinItem -RowLimit $BatchSize | Where-Object {
            $_.DeletedByName -eq $DeletedByName -and $_.DeletedDate -gt $TargetDate
        }

        foreach ($item in $items) {
            $count++
            Write-Host "$($item.Id) :::: $($item.Title) :::: $($item.ItemType) :::: $($item.DirName)"
            try {
                # Comment the next line if you want to skip the restoration and just log the items
                Restore-PnPRecycleBinItem -Identity $item.ID -Force

                
                $logEntry = "$count. Deleted Date: $($item.DeletedDate) ::  Restored item: $($item.Title) from $($item.DirName)"
                Write-Host $logEntry
                $logEntry | Out-File -FilePath $LogFile -Append
            } catch {
                $errorEntry = "$count. Deleted Date: $($item.DeletedDate) :: Failed to restore item: $($item.Title) - $_"
                Write-Warning $errorEntry
                $errorEntry | Out-File -FilePath $LogFile -Append
            }
        }

        Write-Host "Restoration process completed"
    } catch {
        Write-Error "Error during recycle bin item restoration: $_"
        exit 1
    }
}

# === MAIN EXECUTION ===

try {
    $userInput = Get-UserInput

    $batchSize = 999999
    $timeStamp = Get-Date -Format "yyyy-MM-dd-HHmmss"
    $targetDate = (Get-Date).AddDays(-[int]$userInput.numberOfDays).Date
    $logFileName = Join-Path -Path $userInput.logLocation -ChildPath "RestoreFile_$timeStamp.txt"

    Test-LogDirectoryPath -Path $userInput.logLocation
    Connect-ToSharePoint -Tenant $userInput.tenant -ClientID $userInput.clientID -Thumbprint $userInput.thumbprint -SiteURL $userInput.siteURL
    Restore-RecycleBinItems -DeletedByName $userInput.deletedByName -TargetDate $targetDate -BatchSize $batchSize -LogFile $logFileName
} catch {
    Write-Error "Unexpected error occurred: $_"
    exit 1
} finally {
    try {
        Disconnect-PnPOnline
        Write-Host "Disconnected from SharePoint."
    } catch {
        Write-Warning "Failed to disconnect from SharePoint: $_"
    }
}
# End of script
```


## Contributors

| Author(s) |
|-----------|
| Pankaj Badoni |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-restore-multiple-items" aria-hidden="true" />

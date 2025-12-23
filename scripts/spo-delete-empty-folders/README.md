

# Delete Empty Folders in SharePoint Document Library

## Summary

This sample demonstrates how to identify and delete empty folders in a SharePoint document library. The script connects to a specified site, scans the target library (and optional folder path), checks each folder for files and size, and deletes those that are empty. You can run the script in report-only mode to list empty folders without deleting them. This sample is available in both PnP PowerShell and CLI for Microsoft 365.

SharePoint document libraries can accumulate empty folders over time due to user actions, migrations, or automated processes. Empty folders clutter the library, making navigation and management harder for users and administrators. By identifying and removing these folders, the script helps keep the document library organized, improves user experience, and can enhance performance by reducing unnecessary items. The report-only mode also allows safe auditing before deletion, minimizing accidental data loss.

![Example Screenshot](assets/example.png)

# [PnP PowerShell](#tab/pnpps)

```powershell
<#
.SYNOPSIS
    Deletes empty folders from a SharePoint Online document library using only PnP cmdlets.

.DESCRIPTION
    Recursively checks for empty folders (and their subfolders) in a given SharePoint Online document library
    and deletes them if they contain no files or non-empty subfolders.

.PARAMETER SiteUrl
    SharePoint site URL.

.PARAMETER LibraryName
    Name of the document library.

.PARAMETER FolderPath
    Optional. Subfolder path inside the library to start from. If omitted, script checks from the root.

.EXAMPLE
    .\Remove-EmptySPFolders.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Projects" -LibraryName "Shared Documents"
    .\Remove-EmptySPFolders.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Projects" -LibraryName "Shared Documents" -FolderPath "2022/Old"
#>
param (
    [Parameter(Mandatory = $true)]
    [Alias("Url")]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true)]
    [Alias("Library", "List")]
    [string]$LibraryName,

    [Parameter(Mandatory = $false)]
    [Alias("Path", "Folder")]
    [string]$FolderPath,

    [Parameter(Mandatory = $false)]
    [bool]$ReportOnly = $false
)

try {
    $ClientId = "<your-client-id>" # Replace with your Microsoft Entra ID (Azure AD) app client ID
    
    Write-Host "Connecting to $SiteUrl ..." -ForegroundColor Cyan
    Connect-PnPOnline -Url $SiteUrl -Interactive -ClientId $ClientId    

    $fullPath = if ($FolderPath) { "$LibraryName/$FolderPath" } else { $LibraryName }
    Write-Host "Checking for empty folders in: $fullPath" -ForegroundColor Cyan
    $items = Get-PnPFolderItem -FolderSiteRelativeUrl $fullPath -Recursive -ItemType Folder | Where-Object { $_.Name -ne "Forms" }

    Get-PnPList -Identity $LibraryName

    # get current web relative URL
    $currentWebRelativeUrl = (Get-PnPWeb).ServerRelativeUrl
    [System.Array]::Reverse($items)

    $deletedFolders = @()

    # loop through each folder
    foreach ($item in $items) {
        $itemUrl = $item.ServerRelativeUrl -replace "^$currentWebRelativeUrl/", ""
        Write-Host "Processing folder: $itemUrl" -ForegroundColor DarkGray

        $FolderSize = Get-PnPFolderStorageMetric -FolderSiteRelativeUrl $itemUrl | Select-Object -ExpandProperty TotalSize
        $FolderSize = [Math]::Round($FolderSize / 1MB, 2)
        $FolderItemCount = Get-PnPFolderStorageMetric -FolderSiteRelativeUrl $itemUrl | Select-Object -ExpandProperty TotalFileCount
        Write-Host "Folder Size: $FolderSize MB, Item Count: $FolderItemCount" -ForegroundColor DarkGray

        if (($FolderSize -eq 0) -and ($FolderItemCount -eq 0) -and -not ($itemUrl.ToString() -like "*/Forms/*") ) {
            if (-not $ReportOnly) {
                Write-Host "Deleting empty folder: $itemUrl" -ForegroundColor Green

                $nameParam = Split-Path $itemUrl -Leaf
                $folderParam = (Split-Path $itemUrl -Parent) -replace '\\', '/'

                Remove-PnPFolder -Name $nameParam `
                    -Folder $folderParam `
                    -Recycle -Force -ErrorAction Stop
            }
            else {
                Write-Host "Empty folder: $itemUrl" -ForegroundColor Green
            }

            $deletedFolders += [PSCustomObject]@{
                FolderUrl = $itemUrl
            }
        }
    }

    # Export deleted folders to CSV
    $deletedFolders | Export-Csv -Path "DeletedFolders.csv" -NoTypeInformation -Encoding UTF8
}
catch {
    Write-Error "Script failed: $_"
}
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory, HelpMessage = "SharePoint site URL")]
    [string]$SiteUrl,
    
    [Parameter(Mandatory, HelpMessage = "Document library name")]
    [string]$LibraryName,
    
    [Parameter(HelpMessage = "Optional subfolder path to start scanning from")]
    [string]$FolderPath,
    
    [Parameter(HelpMessage = "Report mode - list empty folders without deleting")]
    [switch]$ReportOnly,
    
    [Parameter(HelpMessage = "Export results to CSV file")]
    [switch]$ExportToCsv,
    
    [Parameter(HelpMessage = "Output directory for CSV export")]
    [string]$OutputPath = "."
)

begin {
    Write-Verbose "Verifying CLI for Microsoft 365 login status..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to verify login status. Please run 'm365 login' first."
    }
    
    if ($ExportToCsv -and -not (Test-Path $OutputPath)) {
        throw "Output path does not exist: $OutputPath"
    }
    
    $script:Results = [System.Collections.ArrayList]::new()
    $script:Summary = @{
        TotalFolders = 0
        EmptyFolders = 0
        DeletedFolders = 0
        FailedDeletions = 0
    }
}

process {
    $fullPath = if ($FolderPath) { "$LibraryName/$FolderPath" } else { $LibraryName }
    Write-Verbose "Scanning folders in: $fullPath"
    
    $foldersJson = m365 spo folder list --webUrl $SiteUrl --parentFolderUrl $fullPath --recursive --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve folders: $foldersJson"
    }
    
    $folders = @($foldersJson | ConvertFrom-Json)
    $folders = $folders | Where-Object { $_.Name -ne "Forms" }
    
    $script:Summary.TotalFolders = $folders.Count
    Write-Verbose "Found $($folders.Count) folders to check"
    
    [Array]::Reverse($folders)
    
    foreach ($folder in $folders) {
        Write-Verbose "Processing: $($folder.Name)"
        
        if ($folder.ItemCount -eq 0) {
            $script:Summary.EmptyFolders++
            
            $folderInfo = [PSCustomObject]@{
                FolderPath = $folder.ServerRelativeUrl
                FolderName = $folder.Name
                Status = if ($ReportOnly) { "Found" } else { "Pending" }
                Error = ""
            }
            
            if (-not $ReportOnly) {
                if ($PSCmdlet.ShouldProcess($folder.ServerRelativeUrl, "Delete empty folder")) {
                    try {
                        Write-Verbose "Deleting: $($folder.ServerRelativeUrl)"
                        $removeResult = m365 spo folder remove --webUrl $SiteUrl --url $folder.ServerRelativeUrl --recycle --force 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            throw $removeResult
                        }
                        
                        $folderInfo.Status = "Deleted"
                        $script:Summary.DeletedFolders++
                    }
                    catch {
                        Write-Warning "Failed to delete '$($folder.ServerRelativeUrl)': $_"
                        $folderInfo.Status = "Failed"
                        $folderInfo.Error = $_.ToString()
                        $script:Summary.FailedDeletions++
                    }
                }
            }
            
            [void]$script:Results.Add($folderInfo)
        }
    }
}

end {
    Write-Host "`nSummary:" -ForegroundColor Cyan
    Write-Host "  Total folders scanned: $($script:Summary.TotalFolders)" -ForegroundColor White
    Write-Host "  Empty folders found: $($script:Summary.EmptyFolders)" -ForegroundColor Yellow
    
    if (-not $ReportOnly) {
        $deletedColor = if ($script:Summary.DeletedFolders -gt 0) { "Green" } else { "White" }
        Write-Host "  Folders deleted: $($script:Summary.DeletedFolders)" -ForegroundColor $deletedColor
        
        $failedColor = if ($script:Summary.FailedDeletions -gt 0) { "Red" } else { "White" }
        Write-Host "  Failed deletions: $($script:Summary.FailedDeletions)" -ForegroundColor $failedColor
    }
    
    if ($ExportToCsv) {
        $csvPath = Join-Path $OutputPath "DeletedFolders_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
        $script:Results | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "`nResults exported to: $csvPath" -ForegroundColor Green
    }
    else {
        if ($script:Results.Count -gt 0) {
            Write-Host "`nEmpty Folders:" -ForegroundColor Cyan
            $script:Results | Format-Table -AutoSize
        }
        else {
            Write-Host "`nNo empty folders found." -ForegroundColor Green
        }
    }
}

# Example usage:
# .\Delete-EmptyFolders.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Projects" -LibraryName "Shared Documents"
# .\Delete-EmptyFolders.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Projects" -LibraryName "Shared Documents" -ReportOnly
# .\Delete-EmptyFolders.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Projects" -LibraryName "Shared Documents" -FolderPath "Archive/2023" -ExportToCsv -Verbose
# .\Delete-EmptyFolders.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Projects" -LibraryName "Shared Documents" -WhatIf
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Nanddeep Nachan](https://github.com/nanddeepn) |
| [Smita Nachan](https://github.com/SmitaNachan) |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-delete-empty-folders" aria-hidden="true" />

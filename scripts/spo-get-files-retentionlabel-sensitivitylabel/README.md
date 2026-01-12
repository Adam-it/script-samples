

# Get Files with Retention or Sensitivity Labels in SharePoint Online

## Summary
Understanding the sensitivity and retention labels applied to files in your SharePoint Online sites is essential for maintaining data security and compliance. These labels enable you to manage and protect your data by defining retention periods and handling sensitive information appropriately. This is particularly important for initiatives like the Microsoft 365 Copilot rollout, ensuring that the correct files are stored within the appropriate SharePoint sites. For example, if a SharePoint site is a public Team site, files labeled as confidential should be moved to a private Team site or existing Team site updated from public to private.

Example of file tagged with retention label and sensitivity label:

![Retention Sensitivity Label](./assets/example.png)

## Prerequisites

- The user account that runs the script must have SharePoint Online site administrator access.

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding()]
param(
    [Parameter(HelpMessage = "Filter sites by URL pattern (e.g., 'https://contoso.sharepoint.com/sites/project'). Leave empty to scan all SharePoint sites.")]
    [string]$SiteUrlFilter,
    
    [Parameter(HelpMessage = "Include OneDrive sites in the audit")]
    [switch]$IncludeOneDrive,
    
    [Parameter(HelpMessage = "Output directory path for CSV report")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login status..."
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure login to CLI for Microsoft 365. Please run 'm365 login' first."
    }

    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath -PathType Container)) {
            throw "Output path '$OutputPath' does not exist or is not a directory."
        }
    }

    $script:LabeledFilesCollection = [System.Collections.ArrayList]@()
    $script:Summary = @{
        SitesProcessed = 0
        LibrariesProcessed = 0
        FilesWithLabels = 0
        Failures = 0
    }

    $timestamp = Get-Date -Format "yyyy-MM-dd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "labelsReport-transcript-$timestamp.log"
    Start-Transcript -Path $transcriptPath | Out-Null
    Write-Host "Started transcript logging at: $transcriptPath" -ForegroundColor Cyan

    $ExcludedLibraries = @(
        "Form Templates", "Preservation Hold Library", "Site Assets", "Site Pages", "Images", "Pages", "Settings", "Videos",
        "Site Collection Documents", "Site Collection Images", "Style Library", "AppPages", "Apps for SharePoint", "Apps for Office"
    )
}

process {
    try {
        Write-Host "Retrieving sites from tenant..." -ForegroundColor Cyan
        
        if ($IncludeOneDrive) {
            Write-Verbose "Executing: m365 spo site list --withOneDriveSites --output json"
            $sitesJson = m365 spo site list --withOneDriveSites --output json
        } else {
            Write-Verbose "Executing: m365 spo site list --output json"
            $sitesJson = m365 spo site list --output json
        }
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sites. CLI output: $sitesJson"
        }

        $sites = @($sitesJson | ConvertFrom-Json) | Where-Object { $_.Template -ne 'RedirectSite#0' }
        
        if ($SiteUrlFilter) {
            Write-Host "Applying site URL filter: $SiteUrlFilter" -ForegroundColor Cyan
            $sites = $sites | Where-Object { $_.Url -like "$SiteUrlFilter*" }
            if ($sites.Count -eq 0) {
                Write-Warning "No sites matched the filter pattern '$SiteUrlFilter'. Exiting."
                return
            }
        }
        
        Write-Host "Found $($sites.Count) sites (excluding redirect sites)" -ForegroundColor Green

        foreach ($site in $sites) {
            try {
                $script:Summary.SitesProcessed++
                Write-Host "`nProcessing Site [$($script:Summary.SitesProcessed)/$($sites.Count)]: $($site.Url)" -ForegroundColor Magenta
                Write-Verbose "Site Title: $($site.Title), Template: $($site.Template)"

                Write-Verbose "Executing: m365 spo list list --webUrl $($site.Url) --filter \"BaseTemplate eq 101 and Hidden eq false\" --output json"
                $listsJson = m365 spo list list --webUrl $site.Url --filter "BaseTemplate eq 101 and Hidden eq false" --output json
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve lists for site '$($site.Url)'. CLI: $listsJson"
                    $script:Summary.Failures++
                    continue
                }

                $lists = @($listsJson | ConvertFrom-Json) | Where-Object { $_.Title -notin $ExcludedLibraries }
                Write-Verbose "Found $($lists.Count) document libraries (excluding system libraries)"

                foreach ($library in $lists) {
                    try {
                        $script:Summary.LibrariesProcessed++
                        Write-Host "  Processing Library: $($library.Title)" -ForegroundColor Yellow
                        Write-Verbose "  Library ID: $($library.Id), BaseTemplate: $($library.BaseTemplate)"

                        Write-Verbose "  Executing: m365 spo listitem list --webUrl $($site.Url) --listId $($library.Id) --fields '_ComplianceTag,_DisplayName,FileLeafRef,FileRef,Modified' --output json"
                        $itemsJson = m365 spo listitem list --webUrl $site.Url --listId $library.Id --fields "_ComplianceTag,_DisplayName,FileLeafRef,FileRef,Modified" --output json
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "  Failed to retrieve items for library '$($library.Title)'. CLI: $itemsJson"
                            $script:Summary.Failures++
                            continue
                        }

                        $items = @($itemsJson | ConvertFrom-Json) | Where-Object { $_._ComplianceTag -or $_._DisplayName }
                        Write-Verbose "  Found $($items.Count) files with retention or sensitivity labels"

                        foreach ($item in $items) {
                            $script:Summary.FilesWithLabels++
                            $null = $script:LabeledFilesCollection.Add(([PSCustomObject]@{
                                SiteUrl = $site.Url
                                SiteTitle = $site.Title
                                LibraryTitle = $library.Title
                                FileName = $item.FileLeafRef
                                ServerRelativePath = $item.FileRef
                                RetentionLabel = $item._ComplianceTag
                                SensitivityLabel = $item._DisplayName
                                LastModified = $item.Modified
                            }))
                        }
                    }
                    catch {
                        Write-Warning "  Error processing library '$($library.Title)': $($_.Exception.Message)"
                        $script:Summary.Failures++
                        continue
                    }
                }
            }
            catch {
                Write-Warning "Error processing site '$($site.Url)': $($_.Exception.Message)"
                $script:Summary.Failures++
                continue
            }
        }
    }
    catch {
        Write-Error "Critical error during site retrieval: $($_.Exception.Message)"
        throw
    }
}

end {
    if ($script:LabeledFilesCollection.Count -gt 0) {
        $timestamp = Get-Date -Format "yyyy-MM-dd-HHmmss"
        $csvPath = Join-Path $OutputPath "labelsReport-$timestamp.csv"
        $script:LabeledFilesCollection | Export-Csv -Path $csvPath -NoTypeInformation
        Write-Host "`n==============================================" -ForegroundColor Cyan
        Write-Host "CSV Report Exported" -ForegroundColor Green
        Write-Host "==============================================" -ForegroundColor Cyan
        Write-Host "Path: $csvPath" -ForegroundColor White
        Write-Host "Total Records: $($script:LabeledFilesCollection.Count)" -ForegroundColor White
    } else {
        Write-Host "`n==============================================" -ForegroundColor Yellow
        Write-Host "No Files with Labels Found" -ForegroundColor Yellow
        Write-Host "==============================================" -ForegroundColor Yellow
    }

    Write-Host "`n==============================================" -ForegroundColor Cyan
    Write-Host "Summary" -ForegroundColor Green
    Write-Host "==============================================" -ForegroundColor Cyan
    Write-Host "Sites Processed: " -NoNewline -ForegroundColor White
    Write-Host $script:Summary.SitesProcessed -ForegroundColor Green
    Write-Host "Libraries Processed: " -NoNewline -ForegroundColor White
    Write-Host $script:Summary.LibrariesProcessed -ForegroundColor Green
    Write-Host "Files with Labels: " -NoNewline -ForegroundColor White
    Write-Host $script:Summary.FilesWithLabels -ForegroundColor Green
    Write-Host "Failures: " -NoNewline -ForegroundColor White
    if ($script:Summary.Failures -gt 0) {
        Write-Host $script:Summary.Failures -ForegroundColor Red
    } else {
        Write-Host $script:Summary.Failures -ForegroundColor Green
    }
    Write-Host "==============================================" -ForegroundColor Cyan

    Stop-Transcript | Out-Null
}

# Usage examples:
#
# Example 1: Basic usage - audit all SharePoint sites (excludes OneDrive by default)
# .\Get-FilesRetentionSensitivityLabel.ps1
#
# Example 2: Audit only sites matching a specific URL pattern
# .\Get-FilesRetentionSensitivityLabel.ps1 -SiteUrlFilter "https://contoso.sharepoint.com/sites/project"
#
# Example 3: Include OneDrive sites with custom output path and verbose logging
# .\Get-FilesRetentionSensitivityLabel.ps1 -IncludeOneDrive -OutputPath "C:\Reports" -Verbose
#
# Example 4: Audit specific hub site and all its associated sites
# .\Get-FilesRetentionSensitivityLabel.ps1 -SiteUrlFilter "https://contoso.sharepoint.com/sites/finance"
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

# [PnP PowerShell](#tab/pnpps)
```powershell
param (
    [Parameter(Mandatory = $true)]
    [string] $domain
)

$adminSiteURL = "https://$domain-Admin.SharePoint.com"
$TenantURL = "https://$domain.sharepoint.com"
$dateTime = (Get-Date).ToString("dd-MM-yyyy-hh-ss")
$invocation = (Get-Variable MyInvocation).Value
$directoryPath = Split-Path $invocation.MyCommand.Path
$fileName = "labelsReport" + $dateTime + ".csv"
$outputPath = $directoryPath + "\" + $fileName

Connect-PnPOnline -Url $adminSiteURL -Interactive -WarningAction SilentlyContinue
$adminConnection = Get-PnPConnection

# Exclude certain libraries
$ExcludedLibraries = @(
    "Form Templates", "Preservation Hold Library", "Site Assets", "Site Pages", "Images", "Pages", "Settings", "Videos",
    "Site Collection Documents", "Site Collection Images", "Style Library", "AppPages", "Apps for SharePoint", "Apps for Office"
)

function ReportFileLabels($siteUrl) {
    $report = @()
    Connect-PnPOnline -url $siteUrl -Interactive -WarningAction SilentlyContinue
    $siteconn = Get-PnPConnection
    try {
        $DocLibraries = Get-PnPList -Includes BaseType, Hidden, Title -Connection $siteconn | Where-Object {
            $_.BaseType -eq "DocumentLibrary" -and $_.Hidden -eq $False -and $_.Title -notin $ExcludedLibraries
        }

        $report += $DocLibraries | ForEach-Object {
            Write-Host "Processing Document Library:" $_.Title -ForegroundColor Yellow
            $library = $_

             Get-PnPListItem -List $library.Title -Fields "ID","_ComplianceTag","_DisplayName" -PageSize 1000 -Connection $siteconn | ForEach-Object  {
                if ($_.FieldValues["_DisplayName"] -or $_.FieldValues["_ComplianceTag"]) {
                    [PSCustomObject]@{
                        SiteUrl           = $siteUrl
                        Title             = $_.FieldValues["FileLeafRef"]
                        ServerRelativePath = $_.FieldValues["FileRef"]
                        RetentionLabel    = $_.FieldValues["_ComplianceTag"]
                        SensitivityLabel  = $_.FieldValues["_DisplayName"]
                        LastModified      = $_["Last_x0020_Modified"]
                    }
                }
            }
        }
    } catch {
        Write-Output "An exception was thrown: $($_.Exception.Message)" -ForegroundColor Red
    }
    return $report
}

Get-PnPTenantSite -Filter "Url -like '$TenantURL'" -Connection $adminConnection | Where-Object { $_.Template -ne 'RedirectSite#0' }  | foreach-object {   
    Write-Host "Processing Site:" $_.Url -ForegroundColor Magenta
    $report += ReportFileLabels -siteUrl $_.Url
}

if($report -and $report.Count -gt 0){
    $report | Export-Csv -Path $outputPath -NoTypeInformation
} else {
    Write-Output "No data found" -ForegroundColor Yellow
} 

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Source Credit

Sample first appeared on [PowerShell: Fetch Files with Retention or Sensitivity Labels in SharePoint Online](https://reshmeeauckloo.com/posts/powershell-get-sensitivity-retention-label/)

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it)|
| [Reshmee Auckloo](https://github.com/reshmee011)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-files-retentionlabel-sensitivitylabel" aria-hidden="true" />

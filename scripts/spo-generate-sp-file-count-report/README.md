

# Generate file count report

## Summary

I came across an interesting request on discord, someone wanted to report on the number of files in their SharePoint Online environment using PnP PowerShell or CLI for Microsoft 365.

Not their storage usage, but the number of files, and the number across all sites, libraries and down to the folder level.

This script will generate a report of the number of files in each site, library and folder in your SharePoint Online environment.

## Example output (CSV)

| Type             | Id           | Path                                                                                       | WebUrl       | SiteTitle | DocumentLibraryTitle | DocumentLibraryUrl               | DocumentLibraryId | DirectFolderCount | DirectFilesCount | DirectItemCount | DirectPercentageOfDocLib | DirectPercentageOfSite | TotalFolderCount | TotalFilesCount | TotalItemCount | TotalPercentageOfDocLib | TotalPercentageOfSite |
| ---------------- | ------------ | ------------------------------------------------------------------------------------------ | ------------ | --------- | -------------------- | -------------------------------- | ----------------- | ----------------- | ---------------- | --------------- | ------------------------ | ---------------------- | ---------------- | --------------- | -------------- | ----------------------- | --------------------- |
| Site             |              | __Redacted__                                                                               | __Redacted__ | Intranet  |                      |                                  |                   | 0                 | 0                | 6               | 0%                       | 100%                   | 0                | 0               | 50044          | 0%                      | 100%                  |
| Document Library | __Redacted__ | /sites/sitename/Document Library                                                           | __Redacted__ | Intranet  | Document Library     | /sites/sitename/Document Library | __Redacted__      | 1                 | 41               | 42              | 97.67%                   | 0.08%                  | 1                | 42              | 43             | 100%                    | 0.09%                 |
| Folder           | __Redacted__ | /sites/sitename/Document Library/Open in app                                               | __Redacted__ | Intranet  | Document Library     | /sites/sitename/Document Library | __Redacted__      | 0                 | 1                | 1               | 2.33%                    | 0%                     | 0                | 1               | 1              | 2.33%                   | 0%                    |
| Document Library | __Redacted__ | /sites/sitename/Shared Documents                                                           | __Redacted__ | Intranet  | Documents            | /sites/sitename/Shared Documents | __Redacted__      | 0                 | 3                | 3               | 100%                     | 0.01%                  | 0                | 3               | 3              | 100%                    | 0.01%                 |
| Document Library | __Redacted__ | /sites/sitename/LoadsOfDocuments                                                           | __Redacted__ | Intranet  | LoadsOfDocuments     | /sites/sitename/LoadsOfDocuments | __Redacted__      | 0                 | 49991            | 49991           | 100%                     | 99.89%                 | 0                | 49991           | 49991          | 100%                    | 99.89%                |
| Document Library | __Redacted__ | /sites/sitename/SiteAssets                                                                 | __Redacted__ | Intranet  | Site Assets          | /sites/sitename/SiteAssets       | __Redacted__      | 2                 | 0                | 2               | 28.57%                   | 0%                     | 4                | 3               | 7              | 100%                    | 0.01%                 |
| Folder           | __Redacted__ | /sites/sitename/SiteAssets/SitePages                                                       | __Redacted__ | Intranet  | Site Assets          | /sites/sitename/SiteAssets       | __Redacted__      | 1                 | 0                | 1               | 14.29%                   | 0%                     | 1                | 1               | 2              | 28.57%                  | 0%                    |
| Folder           | __Redacted__ | /sites/sitename/SiteAssets/SitePages/Dan-Toft---Viva-is-coming-home-for-christmas-(almost) | __Redacted__ | Intranet  | Site Assets          | /sites/sitename/SiteAssets       | __Redacted__      | 0                 | 1                | 1               | 14.29%                   | 0%                     | 0                | 1               | 1              | 14.29%                  | 0%                    |
| Folder           | __Redacted__ | /sites/sitename/SiteAssets/Lists                                                           | __Redacted__ | Intranet  | Site Assets          | /sites/sitename/SiteAssets       | __Redacted__      | 1                 | 0                | 1               | 14.29%                   | 0%                     | 1                | 2               | 3              | 42.86%                  | 0.01%                 |
| Folder           | __Redacted__ | /sites/sitename/SiteAssets/Lists/ __Redacted__                                             | __Redacted__ | Intranet  | Site Assets          | /sites/sitename/SiteAssets       | __Redacted__      | 0                 | 2                | 2               | 28.57%                   | 0%                     | 0                | 2               | 2              | 28.57%                  | 0%                    |

## Dictionary for the output

| Column | Description |
| ------ | ----------- |
| Type | The type of the object, Site, Document Library or Folder |
| Id | The ID of the object |
| Path | The server relative path to the object |
| WebUrl | The site collection URL of the object |
| SiteTitle | The title of the site collection |
| DocumentLibraryTitle | The title of the document library |
| DocumentLibraryUrl | The server relative URL to the document library |
| DocumentLibraryId | The ID of the document library |
| DirectFolderCount | The number of folders "directly", or "first layer" under the object |
| DirectFilesCount | The number of files "directly", or "first layer" under the object |
| DirectItemCount | The total number of items (folders and documents) "directly", or "first layer" under the object |
| DirectPercentageOfDocLib | The percentage of the total number of items in the document library, that are stored directly under the current object |
| DirectPercentageOfSite | The percentage of the total number of items in the site collection, that are stored directly under the current object |
| TotalFolderCount | The total number of folders under the object, all sub-folders included |
| TotalFilesCount | The total number of files under the object, all sub-folders included |
| TotalItemCount | The total number of items (folders and documents) under the object, all sub-folders included |
| TotalPercentageOfDocLib | The percentage of the total number of items in the document library, that are stored under the current object |
| TotalPercentageOfSite | The percentage of the total number of items in the site collection, that are stored under the current object |


# [PnP PowerShell](#tab/pnpps)
```powershell

$DOCUMENT_LIBRARY_BASETEMPLATE = 101
$FOLDER_OBJECT_TYPE = 1


$TenantAdminUrl = "https://2v8lc2-admin.sharepoint.com/"
$ClientId = "#####"
$Thumbprint = "#####"


Write-Host "Connecting to Tenant Admin Site..."
Connect-PnPOnline -Url $TenantAdminUrl -Thumbprint $Thumbprint -ClientId $ClientId
$Sites = Get-PnPTenantSite | Where-Object { $_.Template -ne "RedirectSite#0" -and $_.Template -ne "SPSMSITEHOST#0" }


$Report = @()


Write-Host "Processing $($sites.Count) sites..."
foreach ($Site in $Sites) {
    Write-Host "> $($Site.Url)" -ForegroundColor Blue
    $Connection = Connect-PnPOnline -Url $Site.Url -Thumbprint $Thumbprint -ClientId $ClientId -ReturnConnection
    $Lists = Get-PnPList -Connection $Connection | Where-Object { $_.BaseTemplate -eq $DOCUMENT_LIBRARY_BASETEMPLATE -and $_.Hidden -eq $false }
  
    $TotalSiteItemCount = $Lists | ForEach-Object { $_.ItemCount } | Measure-Object -Sum | Select-Object -ExpandProperty Sum
    $Report += [PSCustomObject]@{
        Type                     = "Site"
        Id                       = $Site.Id
        Path                     = $Site.Url
        WebUrl                   = $Site.Url
        SiteTitle                = $Site.Title
        DocumentLibraryTitle     = ""
        DocumentLibraryUrl       = ""
        DocumentLibraryId        = ""
        DirectFolderCount        = 0
        DirectFilesCount         = 0
        DirectItemCount          = $Lists.Count
        DirectPercentageOfDocLib = "0%"
        DirectPercentageOfSite   = "100%"
        TotalFolderCount         = 0
        TotalFilesCount          = 0
        TotalItemCount           = $TotalSiteItemCount
        TotalPercentageOfDocLib  = "0%"
        TotalPercentageOfSite    = "100%"
    }


    foreach ($List in $Lists) {
        write-host "`t> $($List.Title)"

        if ($List.ItemCount -gt 0) {
            $Items = Get-PnPListItem -List $List -Fields "FileRef", "FileDirRef", "ItemChildCount", "FFSObjType", "ID", "FolderChildCount" -PageSize 5000 -Connection $Connection
           
            $Folders = $Items | Where-Object { $_.FieldValues.FSObjType -eq $FOLDER_OBJECT_TYPE } | Sort-Object -Property FileRef
            $Files = $Items | Where-Object { $_.FieldValues.FSObjType -ne $FOLDER_OBJECT_TYPE } | Sort-Object -Property FileRef

            $RootLevelFolderCount = $Folders | Where-Object { $_.FieldValues.FileDirRef -eq $List.RootFolder.ServerRelativeUrl } | Measure-Object | Select-Object -ExpandProperty Count
            $RootLevelFileCount = $Files | Where-Object { $_.FieldValues.FileDirRef -eq $List.RootFolder.ServerRelativeUrl } | Measure-Object | Select-Object -ExpandProperty Count
            $RootLevelItemCount = $RootLevelFolderCount + $RootLevelFileCount

            $Report += [PSCustomObject]@{
                Type                     = "Document Library"
                Id                       = $List.Id
                Path                     = $List.RootFolder.ServerRelativeUrl
                WebUrl                   = $Site.Url
                SiteTitle                = $Site.Title
                DocumentLibraryTitle     = $List.Title
                DocumentLibraryUrl       = $List.RootFolder.ServerRelativeUrl
                DocumentLibraryId        = $List.Id
                DirectFolderCount        = $RootLevelFolderCount
                DirectFilesCount         = $RootLevelFileCount
                DirectItemCount          = $RootLevelItemCount
                DirectPercentageOfDocLib = $RootLevelItemCount -gt 0 ? "$([Math]::Round(($RootLevelItemCount / $List.ItemCount) * 100, 2))%" : "0%"
                DirectPercentageOfSite   = $TotalSiteItemCount -gt 0 ? "$([Math]::Round(($RootLevelItemCount / $TotalSiteItemCount) * 100, 2))%" : "0%"
                TotalFolderCount         = $Folders.Count
                TotalFilesCount          = $Files.Count
                TotalItemCount           = $List.ItemCount
                TotalPercentageOfDocLib  = $List.ItemCount -gt 0 ? "$([Math]::Round(($List.ItemCount / $List.ItemCount) * 100, 2) ?? 0)%" : "0%"
                TotalPercentageOfSite    = $List.ItemCount -gt 0 ? "$([Math]::Round(($List.ItemCount / $TotalSiteItemCount) * 100, 2) ?? 0)%" : "0%"
            }
    




            foreach ($Folder in $Folders) {  
                Write-Host "`t`t> $($Folder.FieldValues.FileRef)"
                
                $TotalSubFolderCount = $Folders | Where-Object { $_.FieldValues.FileRef.StartsWith($folder.FieldValues.FileRef + "/") } | Measure-Object | Select-Object -ExpandProperty Count
                $TotalSubFilesCount = $Files | Where-Object { $_.FieldValues.FileRef.StartsWith($folder.FieldValues.FileRef) } | Measure-Object | Select-Object -ExpandProperty Count
                $TotalItemCount = $TotalSubFolderCount + $TotalSubFilesCount

                $DirectItemCount = ([int]$Folder.FieldValues.ItemChildCount + [int]$Folder.FieldValues.FolderChildCount)

                $Report += [PSCustomObject]@{
                    Type                     = "Folder"
                    Id                       = $Folder.Id
                    Path                     = $Folder.FieldValues.FileRef
                    WebUrl                   = $Site.Url
                    SiteTitle                = $Site.Title
                    DocumentLibraryTitle     = $List.Title
                    DocumentLibraryUrl       = $List.RootFolder.ServerRelativeUrl
                    DocumentLibraryId        = $List.Id
                    DirectFilesCount         = $Folder.FieldValues.ItemChildCount
                    DirectFolderCount        = $Folder.FieldValues.FolderChildCount
                    DirectItemCount          = $DirectItemCount
                    DirectPercentageOfDocLib = $DirectItemCount -gt 0 ? "$([Math]::Round(($DirectItemCount / $List.ItemCount) * 100, 2))%" : "0%"
                    DirectPercentageOfSite   = $DirectItemCount -gt 0 ? "$([Math]::Round(($DirectItemCount / $TotalSiteItemCount) * 100, 2))%" : "0%"
                    TotalFolderCount         = $TotalSubFolderCount
                    TotalFilesCount          = $TotalSubFilesCount
                    TotalItemCount           = $TotalItemCount
                    TotalPercentageOfDocLib  = $TotalItemCount -gt 0 ? "$([Math]::Round(($TotalItemCount / $List.ItemCount) * 100, 2))%" : "0%"
                    TotalPercentageOfSite    = $TotalItemCount -gt 0 ? "$([Math]::Round(($TotalItemCount / $TotalSiteItemCount) * 100, 2))%" : "0%"
                }
            }
        }
    }
}

$Report | Select-Object * | Export-Csv -Path "Report.csv" -NoTypeInformation
Invoke-Item -Path "Report.csv"

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage="SharePoint Admin Center URL (e.g., https://contoso-admin.sharepoint.com)")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$AdminUrl,
    
    [Parameter(HelpMessage="Output path for CSV report (default: current directory)")]
    [string]$OutputPath = (Get-Location).Path,
    
    [Parameter(HelpMessage="Filter sites by URL pattern (e.g., 'project')")]
    [string]$SiteFilter
)

begin {
    $script:ReportCollection = @()
    $script:Summary = @{
        SitesProcessed = 0
        LibrariesProcessed = 0
        TotalFiles = 0
        TotalFolders = 0
        Failures = 0
    }
    
    $timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
    $transcriptPath = "$OutputPath/spo-file-count-$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Starting file count report generation..." -ForegroundColor Cyan
    
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' first."
    }
    
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath)) {
            throw "Output path does not exist: $OutputPath"
        }
    }
}

process {
    Write-Verbose "Retrieving sites from $AdminUrl..."
    
    $sitesJson = if ($SiteFilter) {
        Write-Verbose "Applying site filter: $SiteFilter"
        m365 spo site list --filter "Url -like '$SiteFilter'" --output json
    } else {
        m365 spo site list --output json
    }
    
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve sites from tenant"
    }
    
    $sites = @($sitesJson | ConvertFrom-Json)
    Write-Host "Found $($sites.Count) site(s) to process" -ForegroundColor Green
    
    foreach ($site in $sites) {
        try {
            Write-Host "Processing site: $($site.Title)" -ForegroundColor Yellow
            Write-Verbose "  Site URL: $($site.Url)"
            $script:Summary.SitesProcessed++
            
            $libsJson = m365 spo list list --webUrl $site.Url --query "[?BaseTemplate == \`101\` && !(Hidden)]" --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve lists for site: $($site.Url)"
                $script:Summary.Failures++
                continue
            }
            
            $libraries = @($libsJson | ConvertFrom-Json)
            Write-Verbose "  Found $($libraries.Count) document library(ies)"
            
            $totalSiteFiles = 0
            $totalSiteFolders = 0
            $totalSiteItems = 0
            
            foreach ($lib in $libraries) {
                try {
                    Write-Verbose "    Processing library: $($lib.Title)"
                    $script:Summary.LibrariesProcessed++
                    
                    if ($lib.ItemCount -eq 0) {
                        Write-Verbose "      Library is empty, skipping"
                        continue
                    }
                    
                    $itemsJson = m365 spo listitem list --webUrl $site.Url --listId $lib.Id --fields "FileRef,FSObjType,FileDirRef,ItemChildCount,FolderChildCount" --output json
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to retrieve items for library: $($lib.Title)"
                        $script:Summary.Failures++
                        continue
                    }
                    
                    $items = @($itemsJson | ConvertFrom-Json)
                    $files = @($items | Where-Object { $_.FSObjType -eq 0 })
                    $folders = @($items | Where-Object { $_.FSObjType -eq 1 })
                    
                    Write-Verbose "      Found $($files.Count) file(s) and $($folders.Count) folder(s)"
                    
                    $totalSiteFiles += $files.Count
                    $totalSiteFolders += $folders.Count
                    $totalSiteItems += $lib.ItemCount
                    
                    $script:Summary.TotalFiles += $files.Count
                    $script:Summary.TotalFolders += $folders.Count
                    
                    $rootLevelFiles = @($files | Where-Object { $_.FileDirRef -eq $lib.RootFolder.ServerRelativeUrl })
                    $rootLevelFolders = @($folders | Where-Object { $_.FileDirRef -eq $lib.RootFolder.ServerRelativeUrl })
                    $rootLevelItemCount = $rootLevelFiles.Count + $rootLevelFolders.Count
                    
                    $directPercentageOfDocLib = if ($lib.ItemCount -gt 0) { "{0:F2}%" -f (($rootLevelItemCount / $lib.ItemCount) * 100) } else { "0.00%" }
                    $directPercentageOfSite = if ($totalSiteItems -gt 0) { "{0:F2}%" -f (($rootLevelItemCount / $totalSiteItems) * 100) } else { "0.00%" }
                    $totalPercentageOfSite = if ($totalSiteItems -gt 0) { "{0:F2}%" -f (($lib.ItemCount / $totalSiteItems) * 100) } else { "0.00%" }
                    
                    $script:ReportCollection += [PSCustomObject]@{
                        Type = "Document Library"
                        Id = $lib.Id
                        Path = $lib.RootFolder.ServerRelativeUrl
                        WebUrl = $site.Url
                        SiteTitle = $site.Title
                        DocumentLibraryTitle = $lib.Title
                        DocumentLibraryUrl = $lib.RootFolder.ServerRelativeUrl
                        DocumentLibraryId = $lib.Id
                        DirectFolderCount = $rootLevelFolders.Count
                        DirectFilesCount = $rootLevelFiles.Count
                        DirectItemCount = $rootLevelItemCount
                        DirectPercentageOfDocLib = $directPercentageOfDocLib
                        DirectPercentageOfSite = $directPercentageOfSite
                        TotalFolderCount = $folders.Count
                        TotalFilesCount = $files.Count
                        TotalItemCount = $lib.ItemCount
                        TotalPercentageOfDocLib = "100.00%"
                        TotalPercentageOfSite = $totalPercentageOfSite
                    }
                    
                    foreach ($folder in $folders) {
                        $totalSubFolders = @($folders | Where-Object { $_.FileRef -like "$($folder.FileRef)/*" })
                        $totalSubFiles = @($files | Where-Object { $_.FileRef -like "$($folder.FileRef)/*" })
                        $totalFolderItemCount = $totalSubFolders.Count + $totalSubFiles.Count
                        
                        $directFolderItemCount = ([int]$folder.ItemChildCount + [int]$folder.FolderChildCount)
                        
                        $directPercentageOfDocLib = if ($lib.ItemCount -gt 0) { "{0:F2}%" -f (($directFolderItemCount / $lib.ItemCount) * 100) } else { "0.00%" }
                        $directPercentageOfSite = if ($totalSiteItems -gt 0) { "{0:F2}%" -f (($directFolderItemCount / $totalSiteItems) * 100) } else { "0.00%" }
                        $totalPercentageOfDocLib = if ($lib.ItemCount -gt 0) { "{0:F2}%" -f (($totalFolderItemCount / $lib.ItemCount) * 100) } else { "0.00%" }
                        $totalPercentageOfSite = if ($totalSiteItems -gt 0) { "{0:F2}%" -f (($totalFolderItemCount / $totalSiteItems) * 100) } else { "0.00%" }
                        
                        $script:ReportCollection += [PSCustomObject]@{
                            Type = "Folder"
                            Id = $folder.Id
                            Path = $folder.FileRef
                            WebUrl = $site.Url
                            SiteTitle = $site.Title
                            DocumentLibraryTitle = $lib.Title
                            DocumentLibraryUrl = $lib.RootFolder.ServerRelativeUrl
                            DocumentLibraryId = $lib.Id
                            DirectFilesCount = $folder.ItemChildCount
                            DirectFolderCount = $folder.FolderChildCount
                            DirectItemCount = $directFolderItemCount
                            DirectPercentageOfDocLib = $directPercentageOfDocLib
                            DirectPercentageOfSite = $directPercentageOfSite
                            TotalFolderCount = $totalSubFolders.Count
                            TotalFilesCount = $totalSubFiles.Count
                            TotalItemCount = $totalFolderItemCount
                            TotalPercentageOfDocLib = $totalPercentageOfDocLib
                            TotalPercentageOfSite = $totalPercentageOfSite
                        }
                    }
                }
                catch {
                    Write-Warning "Failed to process library $($lib.Title): $_"
                    $script:Summary.Failures++
                    continue
                }
            }
            
            $siteDirectPercentage = if ($totalSiteItems -gt 0) { "{0:F2}%" -f (($libraries.Count / $totalSiteItems) * 100) } else { "0.00%" }
            
            $script:ReportCollection += [PSCustomObject]@{
                Type = "Site"
                Id = ""
                Path = $site.Url
                WebUrl = $site.Url
                SiteTitle = $site.Title
                DocumentLibraryTitle = ""
                DocumentLibraryUrl = ""
                DocumentLibraryId = ""
                DirectFolderCount = 0
                DirectFilesCount = 0
                DirectItemCount = $libraries.Count
                DirectPercentageOfDocLib = "0.00%"
                DirectPercentageOfSite = "100.00%"
                TotalFolderCount = $totalSiteFolders
                TotalFilesCount = $totalSiteFiles
                TotalItemCount = $totalSiteItems
                TotalPercentageOfDocLib = "0.00%"
                TotalPercentageOfSite = "100.00%"
            }
        }
        catch {
            Write-Warning "Failed to process site $($site.Url): $_"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    $csvPath = "$OutputPath/spo-file-count-$timestamp.csv"
    $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation
    
    Write-Host "`nFile Count Report Summary:" -ForegroundColor Cyan
    Write-Host "  Sites Processed: $($script:Summary.SitesProcessed)" -ForegroundColor Green
    Write-Host "  Libraries Processed: $($script:Summary.LibrariesProcessed)" -ForegroundColor Green
    Write-Host "  Total Files: $($script:Summary.TotalFiles)" -ForegroundColor Green
    Write-Host "  Total Folders: $($script:Summary.TotalFolders)" -ForegroundColor Green
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "  Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
    
    Write-Host "`n  Report saved to: $csvPath" -ForegroundColor Cyan
    Write-Host "  Transcript saved to: $transcriptPath" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Example 1: Generate report for all sites
# .\Generate-FileCountReport.ps1 -AdminUrl "https://contoso-admin.sharepoint.com"

# Example 2: Filter sites by URL pattern
# .\Generate-FileCountReport.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -SiteFilter "project"

# Example 3: Custom output path with verbose output
# .\Generate-FileCountReport.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\\Reports" -Verbose

# Example 4: Generate report for communication sites only
# .\Generate-FileCountReport.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -SiteFilter "sites" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s)                       |
| ------------------------------- |
| [Dan Toft](https://dan-toft.dk) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-generate-sp-file-count-report" aria-hidden="true" />

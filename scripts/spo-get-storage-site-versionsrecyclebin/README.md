

# Get Storage site version Recycle Bin

## Summary

This sample script may help to get a breakdown of storage for files, file versions and recycle bin. I failed to reconcile the site current usage with the total of file size, total file version size and recycle bin size but may help to get insights on storage usage despite I could not explain 30% of storage allocated. In the sample output, a total of around 15 MB was identified to be comprised of total file size, total version size and recycle bin size and could not identify the remaining 6MB.

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as below
- Update the $OutputSite, $OutPutFile and optionally update $ExcludedLibraries to exclude any libraries
# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage = "SharePoint admin center URL (e.g., https://contoso-admin.sharepoint.com)")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$AdminUrl,

    [Parameter(HelpMessage = "Directory path where CSV reports will be saved")]
    [string]$OutputPath = (Get-Location).Path,

    [Parameter(HelpMessage = "OData filter to limit sites (e.g., 'StorageUsage -gt 1000')")]
    [string]$SiteFilter
)

begin {
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' manually."
    }

    if ($PSBoundParameters.ContainsKey('OutputPath') -and -not (Test-Path -Path $OutputPath -PathType Container)) {
        throw "Output path '$OutputPath' does not exist. Please create it first."
    }

    $script:SiteReportCollection = @()
    $script:FileReportCollection = @()

    $script:Summary = @{
        SitesProcessed  = 0
        FilesProcessed  = 0
        LibrariesScanned = 0
        Failures        = 0
    }

    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "StorageReport-Transcript-$timestamp.log"
    Start-Transcript -Path $transcriptPath | Out-Null

    Write-Host "Starting storage analysis..." -ForegroundColor Cyan
}

process {
    try {
        $siteListArgs = @('spo', 'site', 'list', '--output', 'json')
        if ($SiteFilter) {
            $siteListArgs += '--filter'
            $siteListArgs += $SiteFilter
        }

        Write-Verbose "Retrieving sites with filter: $($SiteFilter ? $SiteFilter : 'None')"
        $sitesJson = m365 @siteListArgs 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sites: $sitesJson"
        }
        $sites = @($sitesJson | ConvertFrom-Json)
        Write-Host "Found $($sites.Count) site(s) to analyze" -ForegroundColor Green

        foreach ($site in $sites) {
            try {
                Write-Host "`nProcessing site: $($site.Title) ($($site.Url))" -ForegroundColor Yellow

                $totalFileSize = 0
                $totalVersionSize = 0
                $totalRecycleBinSize = 0
                $fileCount = 0

                Write-Verbose "Retrieving document libraries for $($site.Url)"
                $listsJson = m365 spo list list --webUrl $site.Url --filter "BaseType eq 1 and Hidden eq false" --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve lists for $($site.Url): $listsJson"
                    $script:Summary.Failures++
                    continue
                }
                $libraries = @($listsJson | ConvertFrom-Json)
                Write-Verbose "Found $($libraries.Count) document library(ies)"
                $script:Summary.LibrariesScanned += $libraries.Count

                foreach ($library in $libraries) {
                    Write-Verbose "Processing library: $($library.Title)"

                    try {
                        $itemsJson = m365 spo listitem list --webUrl $site.Url --listTitle $library.Title --fields "Id,FileRef,FSObjType" --output json 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to get items from library '$($library.Title)': $itemsJson"
                            continue
                        }
                        $items = @($itemsJson | ConvertFrom-Json) | Where-Object { $_.FSObjType -eq 0 }
                        Write-Verbose "Found $($items.Count) file(s) in library '$($library.Title)'"

                        foreach ($item in $items) {
                            try {
                                $fileJson = m365 spo file get --webUrl $site.Url --url $item.FileRef --output json 2>&1
                                if ($LASTEXITCODE -ne 0) {
                                    Write-Verbose "Failed to get file metadata for '$($item.FileRef)': $fileJson"
                                    continue
                                }
                                $file = $fileJson | ConvertFrom-Json
                                $fileSize = [int]$file.Length
                                $totalFileSize += $fileSize

                                $versionsJson = m365 spo file version list --webUrl $site.Url --fileUrl $item.FileRef --output json 2>&1
                                if ($LASTEXITCODE -ne 0) {
                                    Write-Verbose "Failed to get versions for '$($item.FileRef)': $versionsJson"
                                    $versionSize = 0
                                    $versionCount = 0
                                } else {
                                    $versions = @($versionsJson | ConvertFrom-Json)
                                    $versionSize = ($versions | Measure-Object -Property Size -Sum).Sum
                                    $totalVersionSize += $versionSize
                                    $versionCount = $versions.Count
                                }

                                $script:FileReportCollection += [PSCustomObject]@{
                                    SiteUrl          = $site.Url
                                    SiteName         = $site.Title
                                    LibraryTitle     = $library.Title
                                    FileRef          = $item.FileRef
                                    FileSizeMB       = [Math]::Round(($fileSize / 1MB), 3)
                                    VersionCount     = $versionCount
                                    TotalVersionSizeMB = [Math]::Round(($versionSize / 1MB), 3)
                                }

                                $fileCount++
                                $script:Summary.FilesProcessed++

                                if ($fileCount % 50 -eq 0) {
                                    Write-Verbose "Processed $fileCount files in site..."
                                }
                            } catch {
                                Write-Warning "Failed to process file '$($item.FileRef)': $($_.Exception.Message)"
                                continue
                            }
                        }
                    } catch {
                        Write-Warning "Failed to process library '$($library.Title)': $($_.Exception.Message)"
                        continue
                    }
                }

                Write-Verbose "Retrieving recycle bin items for $($site.Url)"
                $recycleBinJson = m365 spo site recyclebinitem list --siteUrl $site.Url --output json 2>&1
                if ($LASTEXITCODE -eq 0) {
                    $recycleBinItems = @($recycleBinJson | ConvertFrom-Json)
                    $totalRecycleBinSize = ($recycleBinItems | Measure-Object -Property Size -Sum).Sum
                    Write-Verbose "Recycle bin size: $([Math]::Round(($totalRecycleBinSize / 1MB), 3)) MB"
                } else {
                    Write-Warning "Failed to get recycle bin items: $recycleBinJson"
                }

                $storageUsageMB = [Math]::Round(($site.StorageUsage / 1MB), 3)
                $totalFileSizeMB = [Math]::Round(($totalFileSize / 1MB), 3)
                $totalVersionSizeMB = [Math]::Round(($totalVersionSize / 1MB), 3)
                $totalRecycleBinSizeMB = [Math]::Round(($totalRecycleBinSize / 1MB), 3)
                $accountedStorageMB = $totalFileSizeMB + $totalVersionSizeMB + $totalRecycleBinSizeMB
                $unaccountedStorageMB = [Math]::Round(($storageUsageMB - $accountedStorageMB), 3)

                $script:SiteReportCollection += [PSCustomObject]@{
                    SiteUrl                = $site.Url
                    SiteName               = $site.Title
                    StorageUsageMB         = $storageUsageMB
                    TotalFileSizeMB        = $totalFileSizeMB
                    TotalVersionSizeMB     = $totalVersionSizeMB
                    TotalRecycleBinSizeMB  = $totalRecycleBinSizeMB
                    AccountedStorageMB     = $accountedStorageMB
                    UnaccountedStorageMB   = $unaccountedStorageMB
                }

                $script:Summary.SitesProcessed++
                Write-Host "Completed: $fileCount file(s) analyzed, Storage: $storageUsageMB MB (Accounted: $accountedStorageMB MB, Unaccounted: $unaccountedStorageMB MB)" -ForegroundColor Green
            } catch {
                Write-Warning "Failed to process site '$($site.Url)': $($_.Exception.Message)"
                $script:Summary.Failures++
                continue
            }
        }
    } catch {
        Write-Error "Critical error during site processing: $($_.Exception.Message)"
        throw
    }
}

end {
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $siteReportPath = Join-Path $OutputPath "SiteStorageReport-$timestamp.csv"
    $fileReportPath = Join-Path $OutputPath "FileStorageReport-$timestamp.csv"

    if ($script:SiteReportCollection.Count -gt 0) {
        $script:SiteReportCollection | Export-Csv -Path $siteReportPath -NoTypeInformation -Force
        Write-Host "`nSite report exported to: " -NoNewline
        Write-Host $siteReportPath -ForegroundColor Cyan
    }

    if ($script:FileReportCollection.Count -gt 0) {
        $script:FileReportCollection | Export-Csv -Path $fileReportPath -NoTypeInformation -Force
        Write-Host "File report exported to: " -NoNewline
        Write-Host $fileReportPath -ForegroundColor Cyan
    }

    Write-Host "`n========== Storage Analysis Summary ==========" -ForegroundColor Cyan
    Write-Host "Sites Processed   : " -NoNewline
    Write-Host $script:Summary.SitesProcessed -ForegroundColor Green
    Write-Host "Libraries Scanned : " -NoNewline
    Write-Host $script:Summary.LibrariesScanned -ForegroundColor Green
    Write-Host "Files Analyzed    : " -NoNewline
    Write-Host $script:Summary.FilesProcessed -ForegroundColor Green
    Write-Host "Failures          : " -NoNewline
    Write-Host $script:Summary.Failures -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "=============================================" -ForegroundColor Cyan

    Stop-Transcript | Out-Null
}

# Basic usage - analyze all sites
# .\Get-StorageSiteVersionsRecycleBin.ps1 -AdminUrl "https://contoso-admin.sharepoint.com"

# Filter sites with storage > 1000 MB
# .\Get-StorageSiteVersionsRecycleBin.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -SiteFilter "StorageUsage -gt 1000"

# Specify custom output path with verbose logging
# .\Get-StorageSiteVersionsRecycleBin.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\Reports" -Verbose

# Analyze specific site by title
# .\Get-StorageSiteVersionsRecycleBin.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -SiteFilter "Title eq 'Project Site'"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***


# [PnP PowerShell](#tab/pnpps)

```powershell

$SharePointAdminSiteURL = "https://contoso-admin.sharepoint.com"
$conn = Connect-PnPOnline -Url $SharePointAdminSiteURL -Interactive

# Set Variables
$dateTime = (Get-Date).toString("dd-MM-yyyy")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "\SiteStoragReport-" + $dateTime + ".csv"
$OutputSite = $directorypath + $fileName
$fileName = "\FileStorageReport-" + $dateTime + ".csv"
$OutPutFile = $directorypath + $fileName

$arraySite = New-Object System.Collections.ArrayList
$arrayFile = New-Object System.Collections.ArrayList

#Exclude certain libraries
#$ExcludedLibraries = @("Form Templates", "Preservation Hold Library", "Site Assets", "Site Pages", "Images", "Pages", "Settings", "Videos", "Site Collection Documents", "Site Collection Images", "Style Library", "AppPages", "Apps for SharePoint", "Apps for Office")

function ReportStorageVersions($site) {
    try {
        $fileSizes = @(); 
        $fileSize = 0 
        $TotalVersionSize = 0
        $DocLibraries = Get-PnPList -Includes BaseType, Hidden, Title -Connection $siteconn | Where-Object { $_.BaseType -eq "DocumentLibrary" -and $_.Hidden -eq $False -and $_.Title -notin $ExcludedLibraries }
        $DocLibraries | ForEach-Object {
            Write-host "Processing Document Library:" $_.Title -f Yellow
            $library = $_
            $listItems = Get-PnPListItem -List $library.Title -Fields "ID" -PageSize 1000 -Connection $siteconn

            #Get file zize
            $listItems | ForEach-Object {
                $listitem = $_
                $fileVersionSize = 0
                $file = Get-PnPFile -Url $listitem["FileRef"] -AsFileObject -ErrorAction SilentlyContinue -Connection $siteconn 

                if ($file) {
                    $fileSize += $file.Length          
                    $elementFile = "" | Select-Object SiteUrl, siteName, siteStorage, FileRef,FileSize,TotalVersionSize,VersionCount,StartTime, EndTime
                    $elementFile.SiteUrl = $site.Url
                    $elementFile.siteName = $site.Title
                    $elementFile.siteStorage = "$siteStorage MB"
                    $elementFile.StartTime = (Get-Date).toString("dd-MM-yyyy HH:mm:ss")
                    $elementFile.FileRef  =   $listitem["FileRef"]
                    $fileversions = Get-PnPFileVersion -Url $listitem["FileRef"] -Connection $siteconn
                    if ($fileversions) {
                        # Calculate the total version size
                        $fileVersionSize = $fileversions | Measure-Object -Property Size -Sum | Select-Object -ExpandProperty Sum                                                   
                    }

                    $elementFile.FileSize = "$([Math]::Round(($file.Length/1MB),3)) MB" 
                    $elementFile.TotalVersionSize = "$([Math]::Round(($fileVersionSize/1MB),3)) MB"
                    $elementFile.VersionCount = $fileversions.Count
                    $totalVersionSize += $fileVersionSize
                    $elementFile.EndTime = (Get-Date).toString("dd-MM-yyyy HH:mm:ss")
                    $arrayFile.Add($elementFile) | Out-Null 
                }        
            }
        }
        $fileSizes += $fileSize
        $fileSizes += $totalVersionSize   

        return $fileSizes
    }
    catch {
        Write-Output "An exception was thrown: $($_.Exception.Message)" -ForegroundColor Red
    } 
}

# Get total storage use for the site collection, amend query to run reports against site collection(s), e.g. filter by $_.StorageUsageCurrent -gt 10000
Get-PnPTenantSite -Connection $conn | Where-Object { ($_.Template -eq "GROUP#0" -or $_.Template -eq "SITEPAGEPUBLISHING#0") -and $_.Title -eq "Company 311"} | ForEach-Object {
    $site = $_
    $siteStorage = $site.StorageUsageCurrent
    #$siteStorage = $siteStorage/1024l
    #$siteStorage = [Math]::Round($siteStorage, 2)
    Write-Host "Site storage: $siteStorage MB"
    $siteconn = Connect-PnPOnline -Url $site.Url -Interactive -ReturnConnection

    $element = "" | Select-Object SiteUrl, siteName, siteStorage, FileSize, StartTime,TotalVersionSize, RecycleBinSize,EndTime
    $element.SiteUrl = $site.Url
    $element.siteName = $site.Title
    $element.siteStorage = "$siteStorage MB"
    $element.StartTime = (Get-Date).toString("dd-MM-yyyy HH:mm:ss")
    $FileSizeVersions = ReportStorageVersions -site $site
    $element.FileSize = "$([Math]::Round(($FileSizeVersions[0]/1MB),3)) MB" 
    $element.TotalVersionSize = "$([Math]::Round(($FileSizeVersions[1]/1MB),3)) MB"
    $RecycleBinItemsSize = Get-PnPRecycleBinItem -Connection $siteconn | Measure-Object -Property Size -Sum | Select-Object -ExpandProperty Sum
    $element.RecycleBinSize = "$([Math]::Round(($RecycleBinItemsSize/1MB),3)) MB"
    $element.EndTime = (Get-Date).toString("dd-MM-yyyy HH:mm:ss")

    $arraySite.Add($element) | Out-Null 
}  

$arraySite | Export-Csv -Path $OutputSite -NoTypeInformation -Force 
$arrayFile | Export-Csv -Path $OutputFile -NoTypeInformation -Force

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-storage-site-versionsrecyclebin" aria-hidden="true" />
| [Adam Wójcik](https://github.com/Adam-it)|
| [Reshmee Auckloo](https://github.com/reshmee011)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-storage-site-versionsrecyclebin" aria-hidden="true" />

# SharePoint Online - Export Duplicate Files

## Summary

This sample contains PowerShell scripts that loop through all the files in a SharePoint Online Tenant and compare file hashes to identify duplicate files. The scripts generate a report without deleting files, allowing you to review duplicates for manual cleanup.

The PnP PowerShell version uses Microsoft Graph API via `Invoke-PnPGraphMethod`. The CLI for Microsoft 365 version (v11.4.0+) uses `m365 request` for Graph API calls and `m365 spo site list` for site enumeration.

**NOTE** the proper way to do this would be using [Microsoft Graph Data Connect](https://learn.microsoft.com/en-us/graph/data-connect-concept-overview), but I'm cheap, and only needed it on my development tenant, so I wrote this script instead, but the concept remains the same.

**NOTE** I Ran this using delegated permissions, but if you want the full overview of all files, you should run this using application permissions, as delegated permissions will only return files that the user has access to.

# [PnP PowerShell](#tab/pnpps)

```powershell
$ErrorActionPreference = "Stop"
$SharePointRootSiteUrl = "http://<tenant>.sharepoint.com/"


Connect-PnPOnline -Interactive -Url $SharePointRootSiteUrl -ClientId "<ClientId>";

$allFiles = New-Object System.Collections.ArrayList;


$sites = Invoke-PnPGraphMethod -Url "https://graph.microsoft.com/v1.0/sites/?`$search=`"http*`"&`$select=id,webUrl,displayName&`$top=100" -All;

foreach ($site in $sites.value) {
    Write-Host "> Site: $($site.displayName) - ($($site.webUrl))"
    $drives = Invoke-PnPGraphMethod -Url "https://graph.microsoft.com/v1.0/sites/$($site.id)/drives?`$select=id,webUrl,name&`$top=100" -All;

    foreach ($drive in $drives.value) {
        Write-Host "`t> Drive: $($drive.name) - ($($drive.webUrl))";

        ## Would've loved to use a $select=file,id,webUrl,size,name but that breaks for some reason when using PnP PowerShell
        $files = Invoke-PnPGraphMethod -Url "https://graph.microsoft.com/v1.0/sites/$($site.id)/drives/$($drive.id)/items?`$filter=file ne null" -All;

        foreach ($file in $files.value | Where-Object { $_.file -ne $null }) {
            Write-Host "`t`t>File: $($file.name)";

            $allFiles.Add([PSCustomObject]@{
                    SiteId     = $site.id
                    DriveId    = $drive.id
                    FileId     = $file.id
                    FileName   = $file.name
                    FileWebUrl = $file.webUrl
                    FileSize   = $file.size
                    FileHash   = $file.file.hashes.quickXorHash
                }) | Out-Null
        }
        Write-Host "`t> Finished processing files in drive: $($drive.name)"
    }
    Write-Host "> Finished processing drives in site: $($site.displayName)"   
}

Write-Host "Finished loading all files"

$grouped = $allFiles | Where-Object {$null -ne $_.FileHash} | Group-Object -Property FileHash | Where-Object { $_.Count -gt 1 } | Sort-Object -Property Count -Descending;

foreach($group in $grouped){
    Write-Host "Duplicate files with hash: $($group.Name)"
    foreach($file in $group.Group){
        Write-Host "`t> $($file.FileName) - $($file.FileWebUrl)"
    }
    Write-Host ""
}


```

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory = $false, HelpMessage = "Folder path for CSV export")]
    [ValidateScript({ Test-Path $_ -PathType Container })]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    
    $script:AllFiles = [System.Collections.ArrayList]::new()
    $script:Summary = @{
        SitesProcessed = 0
        DrivesProcessed = 0
        FilesProcessed = 0
        DuplicateGroups = 0
        Failures = 0
    }
    
    $logPath = Join-Path $OutputPath "DuplicateFiles_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    Start-Transcript -Path $logPath
    
    Write-Host "Starting duplicate file detection across tenant..." -ForegroundColor Cyan
}

process {
    try {
        Write-Host "`nGetting all sites in tenant..." -ForegroundColor Yellow
        $sitesJson = m365 spo site list --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sites"
        }
        $sites = @($sitesJson | ConvertFrom-Json)
        Write-Host "  Found $($sites.Count) sites" -ForegroundColor Green
        
        $siteIndex = 0
        foreach ($site in $sites) {
            $siteIndex++
            Write-Progress -Activity "Processing Sites" -Status "$($site.Title) ($siteIndex of $($sites.Count))" -PercentComplete (($siteIndex / $sites.Count) * 100)
            
            try {
                Write-Verbose "Getting drives for site: $($site.Title)"
                $drivesJson = m365 request --url "https://graph.microsoft.com/v1.0/sites/$($site.Id)/drives?`$select=id,webUrl,name" --output json
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to get drives for site: $($site.Title)"
                    $script:Summary.Failures++
                    continue
                }
                
                $drivesResponse = $drivesJson | ConvertFrom-Json
                $drives = @($drivesResponse.value)
                
                if ($drives.Count -eq 0) {
                    Write-Verbose "  No drives found in site: $($site.Title)"
                    continue
                }
                
                foreach ($drive in $drives) {
                    Write-Verbose "  Processing drive: $($drive.name)"
                    
                    try {
                        Write-Verbose "    Getting all files from drive: $($drive.name)"
                        $filesUrl = "https://graph.microsoft.com/v1.0/sites/$($site.Id)/drives/$($drive.id)/items?`$filter=file ne null&`$select=id,webUrl,size,name,file"
                        $filesJson = m365 request --url $filesUrl --output json
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "    Failed to get files for drive: $($drive.name)"
                            $script:Summary.Failures++
                            continue
                        }
                        
                        $filesResponse = $filesJson | ConvertFrom-Json
                        $files = @($filesResponse.value)
                        
                        while ($filesResponse.'@odata.nextLink') {
                            Write-Verbose "      Fetching next page..."
                            $filesJson = m365 request --url $filesResponse.'@odata.nextLink' --output json
                            if ($LASTEXITCODE -eq 0) {
                                $filesResponse = $filesJson | ConvertFrom-Json
                                $files += @($filesResponse.value)
                            } else {
                                Write-Warning "      Failed to fetch next page"
                                break
                            }
                        }
                        
                        foreach ($file in $files) {
                            if ($file.file -and $file.file.hashes.quickXorHash) {
                                [void]$script:AllFiles.Add([PSCustomObject]@{
                                    SiteId = $site.Id
                                    SiteTitle = $site.Title
                                    DriveId = $drive.id
                                    DriveName = $drive.name
                                    FileId = $file.id
                                    FileName = $file.name
                                    FileWebUrl = $file.webUrl
                                    FileSize = $file.size
                                    FileHash = $file.file.hashes.quickXorHash
                                })
                                $script:Summary.FilesProcessed++
                            }
                        }
                        
                        $script:Summary.DrivesProcessed++
                        Write-Verbose "    Processed $($files.Count) files from drive: $($drive.name)"
                    }
                    catch {
                        Write-Warning "  Error processing drive '$($drive.name)': $_"
                        $script:Summary.Failures++
                        continue
                    }
                }
                
                $script:Summary.SitesProcessed++
            }
            catch {
                Write-Warning "Error processing site '$($site.Title)': $_"
                $script:Summary.Failures++
                continue
            }
        }
        
        Write-Progress -Activity "Processing Sites" -Completed
    }
    catch {
        Write-Host "`nFailed to process tenant: $_" -ForegroundColor Red
        throw
    }
}

end {
    Write-Host "`nAnalyzing files for duplicates..." -ForegroundColor Yellow
    
    $duplicateGroups = $script:AllFiles | 
        Where-Object { $null -ne $_.FileHash } | 
        Group-Object -Property FileHash | 
        Where-Object { $_.Count -gt 1 } | 
        Sort-Object -Property Count -Descending
    
    $script:Summary.DuplicateGroups = $duplicateGroups.Count
    
    if ($duplicateGroups.Count -gt 0) {
        Write-Host "`n=== Duplicate Files Found ==="  -ForegroundColor Cyan
        
        foreach ($group in $duplicateGroups) {
            Write-Host "`nHash: $($group.Name) ($($group.Count) duplicates)" -ForegroundColor Yellow
            foreach ($file in $group.Group) {
                Write-Host "  $($file.FileName) - $($file.FileWebUrl)" -ForegroundColor White
            }
        }
        
        $csvPath = Join-Path $OutputPath "DuplicateFiles_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
        $duplicateData = $duplicateGroups | ForEach-Object {
            foreach ($file in $_.Group) {
                [PSCustomObject]@{
                    Hash = $_.Name
                    DuplicateCount = $_.Count
                    SiteTitle = $file.SiteTitle
                    DriveName = $file.DriveName
                    FileName = $file.FileName
                    FileWebUrl = $file.FileWebUrl
                    FileSizeBytes = $file.FileSize
                }
            }
        }
        
        $duplicateData | Export-Csv -Path $csvPath -NoTypeInformation
        Write-Host "`nDuplicate files report exported to: " -NoNewline -ForegroundColor Green
        Write-Host $csvPath -ForegroundColor Cyan
    } else {
        Write-Host "`nNo duplicate files found!" -ForegroundColor Green
    }
    
    Write-Host "`n=== Duplicate File Detection Summary ===" -ForegroundColor Cyan
    Write-Host "Sites Processed: " -NoNewline
    Write-Host $script:Summary.SitesProcessed -ForegroundColor Green
    Write-Host "Drives Processed: " -NoNewline
    Write-Host $script:Summary.DrivesProcessed -ForegroundColor Green
    Write-Host "Files Processed: " -NoNewline
    Write-Host $script:Summary.FilesProcessed -ForegroundColor Green
    Write-Host "Duplicate Groups: " -NoNewline
    if ($script:Summary.DuplicateGroups -gt 0) {
        Write-Host $script:Summary.DuplicateGroups -ForegroundColor Yellow
    } else {
        Write-Host $script:Summary.DuplicateGroups -ForegroundColor Green
    }
    Write-Host "Failures: " -NoNewline
    if ($script:Summary.Failures -gt 0) {
        Write-Host $script:Summary.Failures -ForegroundColor Red
    } else {
        Write-Host $script:Summary.Failures -ForegroundColor Green
    }
    Write-Host "Log File: " -NoNewline
    Write-Host $logPath -ForegroundColor Cyan
    Write-Host "=========================================`n" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Example 1: Run duplicate detection with default output path
# .\Export-DuplicateFiles.ps1

# Example 2: Run with custom output path
# .\Export-DuplicateFiles.ps1 -OutputPath "C:\Reports"

# Example 3: Run with verbose output for troubleshooting
# .\Export-DuplicateFiles.ps1 -OutputPath "C:\Reports" -Verbose

# Example 4: Run and view log file immediately
# .\Export-DuplicateFiles.ps1; Get-Content .\DuplicateFiles_*.log -Tail 50
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s)                       |
| ------------------------------- |
| [Dan Toft](https://Dan-toft.dk) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/template-script-submission" aria-hidden="true" />

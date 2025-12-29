

# Download all documents from all document libraries in a site, including version history

## Summary

This script downloads all documents from all document libraries in a SharePoint site, optionally including version history. The documents are saved in a local folder structure that matches the library structure. Available in both PnP PowerShell and CLI for Microsoft 365 versions.

![Example Screenshot](assets/example01.png)

This screenshot shows the script in action downloading documents from a site.

![Example Screenshot](assets/example02.png)

This screenshot shows the folder structure created by the script after the documents have been downloaded.

### Prerequisites

- **PnP PowerShell**: Requires an active connection via `Connect-PnPOnline`
- **CLI for Microsoft 365**: Requires login via `m365 login`

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "Site URL to download files from")]
    [string]$SiteUrl,
    
    [Parameter(Mandatory, HelpMessage = "Local directory path to save downloaded files")]
    [string]$DownloadPath,
    
    [Parameter(HelpMessage = "Include version history for all files")]
    [switch]$IncludeVersions,
    
    [Parameter(HelpMessage = "Overwrite existing files in local directory")]
    [switch]$Overwrite,
    
    [Parameter(HelpMessage = "Comma-separated list of library titles to exclude (default: system libraries)")]
    [string[]]$ExcludedLibraries = @(
        "Form Templates", 
        "Preservation Hold Library", 
        "Site Assets", 
        "Images", 
        "Pages", 
        "Settings", 
        "Videos", 
        "Style Library", 
        "AppPages", 
        "Apps for SharePoint"
    )
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure CLI for Microsoft 365 login. Please run 'm365 login' manually."
    }

    if (-not (Test-Path -Path $DownloadPath)) {
        Write-Verbose "Creating download directory: $DownloadPath"
        New-Item -Path $DownloadPath -ItemType Directory | Out-Null
    }

    Write-Verbose "Retrieving all document libraries from site: $SiteUrl"
    $librariesJson = m365 spo list list --webUrl $SiteUrl --filter "BaseTemplate eq 101" --properties "Title,RootFolder/ServerRelativeUrl" --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve document libraries from '$SiteUrl'. CLI: $librariesJson"
    }

    $allLibraries = @($librariesJson | ConvertFrom-Json)
    $libraries = $allLibraries | Where-Object { $_.Title -notin $ExcludedLibraries }

    Write-Host "`n Found $($allLibraries.Count) document libraries ($($libraries.Count) after exclusions)" -ForegroundColor Cyan
    Write-Host " Excluded libraries: $($ExcludedLibraries -join ', ')`n" -ForegroundColor Yellow

    $script:Summary = @{
        LibrariesProcessed = 0
        FilesDownloaded = 0
        FilesSkipped = 0
        FilesAlreadyExist = 0
        VersionsDownloaded = 0
        VersionsSkipped = 0
        Failures = 0
    }
}

process {
    $libraryCounter = 0
    $totalLibraries = $libraries.Count

    foreach ($library in $libraries) {
        $libraryCounter++
        Write-Progress -Activity "Processing Document Libraries" -Status "Library $libraryCounter of $totalLibraries : $($library.Title)" -PercentComplete (($libraryCounter / $totalLibraries) * 100)

        $script:Summary.LibrariesProcessed++

        try {
            $folderUrl = $library.RootFolder.ServerRelativeUrl
            Write-Verbose "  Retrieving files from library: $($library.Title)"
            $filesJson = m365 spo file list --webUrl $SiteUrl --folderUrl "$folderUrl" --recursive --output json 2>&1
            
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve files from library '$($library.Title)'. CLI: $filesJson"
                $script:Summary.Failures++
                continue
            }

            $files = @($filesJson | ConvertFrom-Json)
            
            if ($files.Count -eq 0) {
                Write-Verbose "  Library '$($library.Title)' is empty"
                continue
            }
            
            Write-Verbose "  Processing $($files.Count) files from library '$($library.Title)'"
            $fileCounter = 0
            $totalFiles = $files.Count

            foreach ($file in $files) {
                $fileCounter++
                Write-Progress -Activity "Downloading Files from $($library.Title)" -Status "File $fileCounter of $totalFiles : $($file.Name)" -PercentComplete (($fileCounter / $totalFiles) * 100) -Id 1 -ParentId 0

                try {
                    $fileServerRelativeUrl = $file.ServerRelativeUrl
                    $fileName = $file.Name
                    
                    $relativePath = $fileServerRelativeUrl -replace [regex]::Escape($folderUrl), ''
                    $relativePath = $relativePath.TrimStart('/')
                    $fileDirectory = Split-Path $relativePath -Parent
                    
                    if ($fileDirectory) {
                        $localDirectory = Join-Path $DownloadPath $fileDirectory.Replace('/', '\')
                    } else {
                        $localDirectory = $DownloadPath
                    }

                    if (-not (Test-Path -Path $localDirectory)) {
                        New-Item -Path $localDirectory -ItemType Directory -Force | Out-Null
                    }

                    $localFilePath = Join-Path $localDirectory $fileName

                    if ((Test-Path -Path $localFilePath) -and -not $Overwrite) {
                        Write-Verbose "  File already exists: $fileName (skipping)"
                        $script:Summary.FilesAlreadyExist++
                        continue
                    }

                    if ($PSCmdlet.ShouldProcess($fileServerRelativeUrl, "Download file")) {
                        Write-Verbose "  Downloading: $fileName"
                        $downloadResult = m365 spo file get --webUrl $SiteUrl --url $fileServerRelativeUrl --asFile --path $localDirectory 2>&1
                        
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to download file '$fileName'. CLI: $downloadResult"
                            $script:Summary.Failures++
                            continue
                        }
                       
                        Write-Verbose "  Downloaded: $fileName"
                        $script:Summary.FilesDownloaded++

                        if ($IncludeVersions) {
                            try {
                                Write-Verbose "    Retrieving versions for: $fileName"
                                $versionsJson = m365 spo file version list --webUrl $SiteUrl --fileUrl $fileServerRelativeUrl --output json 2>&1
                                
                               if ($LASTEXITCODE -ne 0) {
                                   Write-Warning "Failed to retrieve versions for '$fileName'. CLI: $versionsJson"
                               }

                               $versions = @($versionsJson | ConvertFrom-Json)
                               
                                if ($LASTEXITCODE -eq 0 -and $versions.Count -gt 0) {
                                    Write-Verbose "    Found $($versions.Count) versions for '$fileName'"
                                    
                                    foreach ($version in $versions) {
                                        try {
                                            $versionLabel = $version.VersionLabel
                                            $fileBaseName = [System.IO.Path]::GetFileNameWithoutExtension($fileName)
                                            $fileExtension = [System.IO.Path]::GetExtension($fileName)
                                            $versionFileName = "${fileBaseName}_${versionLabel}${fileExtension}"
                                            $versionFilePath = Join-Path $localDirectory $versionFileName

                                           if ((Test-Path -Path $versionFilePath) -and -not $Overwrite) {
                                               Write-Verbose "    Version file already exists: $versionFileName (skipping)"
                                                $script:Summary.VersionsSkipped++
                                               continue
                                            }

                                            if ($PSCmdlet.ShouldProcess($versionFileName, "Download version")) {
                                                $versionUrl = $version.Url
                                                Write-Verbose "    Downloading version: $versionLabel"
                                                
                                                $versionDownloadResult = m365 spo file get --webUrl $SiteUrl --url "/$versionUrl" --asFile --path $localDirectory 2>&1
                                                
                                               if ($LASTEXITCODE -ne 0) {
                                                   Write-Warning "Failed to download version '$versionLabel' of '$fileName'. CLI: $versionDownloadResult"
                                                   $script:Summary.Failures++
                                               }

                                               $downloadedVersionFile = Join-Path $localDirectory ([System.IO.Path]::GetFileName($versionUrl))
                                                
                                                if ($LASTEXITCODE -eq 0) {
                                                    if (Test-Path $downloadedVersionFile) {
                                                        Rename-Item -Path $downloadedVersionFile -NewName $versionFileName -Force
                                                        
                                                        if (Test-Path $versionFilePath) {
                                                            Write-Verbose "    Downloaded version: $versionLabel"
                                                            $script:Summary.VersionsDownloaded++
                                                        } else {
                                                            Write-Warning "Version file not found after rename: $versionFileName"
                                                            $script:Summary.Failures++
                                                        }
                                                    } else {
                                                        Write-Warning "Downloaded version file not found: $versionUrl"
                                                        $script:Summary.Failures++
                                                    }
                                                }
                                            }
                                        }
                                        catch {
                                            Write-Warning "Error processing version '$($version.VersionLabel)' of '$fileName': $_"
                                            $script:Summary.Failures++
                                            continue
                                        }
                                    }
                                } else {
                                    Write-Verbose "    No versions available for file: $fileName"
                                }
                            }
                            catch {
                                Write-Warning "Error retrieving versions for '$fileName': $_"
                                $script:Summary.Failures++
                                continue
                            }
                        }
                    } else {
                        $script:Summary.FilesSkipped++
                    }
                }
                catch {
                    Write-Warning "Error processing file '$($file.Name)': $_"
                    $script:Summary.Failures++
                    continue
                }
            }
       }
       catch {
           Write-Warning "Error processing library '$($library.Title)': $_"
           $script:Summary.Failures++
           continue
       }
        
        Write-Progress -Activity "Downloading Files from $($library.Title)" -Id 1 -Completed
    }
    
    Write-Progress -Activity "Processing Document Libraries" -Completed
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host " Download Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "  Libraries Processed : $($Summary.LibrariesProcessed)" -ForegroundColor White
    Write-Host "  Files Downloaded    : $($Summary.FilesDownloaded)" -ForegroundColor Green
    Write-Host "  Files Already Exist : $($Summary.FilesAlreadyExist)" -ForegroundColor Yellow
    Write-Host "  Files Skipped       : $($Summary.FilesSkipped)" -ForegroundColor Yellow
    
   if ($IncludeVersions) {
       Write-Host "  Versions Downloaded : $($Summary.VersionsDownloaded)" -ForegroundColor Green
        Write-Host "  Versions Skipped    : $($Summary.VersionsSkipped)" -ForegroundColor Yellow
   }
    
    $failureColor = if ($Summary.Failures -gt 0) { "Red" } else { "Green" }
    Write-Host "  Failures            : $($Summary.Failures)" -ForegroundColor $failureColor
    Write-Host "========================================`n" -ForegroundColor Cyan
}

# Example 1: Download all files from a site
# .\Download-AllDocLibs.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -DownloadPath "C:\Backup"

# Example 2: Download with version history
# .\Download-AllDocLibs.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -DownloadPath "C:\Backup" -IncludeVersions

# Example 3: Overwrite existing files
# .\Download-AllDocLibs.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -DownloadPath "C:\Backup" -Overwrite

# Example 4: Preview downloads with WhatIf
# .\Download-AllDocLibs.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -DownloadPath "C:\Backup" -WhatIf -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

<#
.SYNOPSIS
Downloads files from all SharePoint document libraries in a tenant to a local directory. Requires PnP PowerShell 2.x and and existing connection to the tenant with Connect-PnPOnline.

.DESCRIPTION
This script connects to a SharePoint tenant using PnP PowerShell, iterates through all document libraries, and downloads files to a specified local directory. It can optionally overwrite existing files and download all versions of the files.

.PARAMETER DownloadPath
Specifies the local directory where files will be saved.

.PARAMETER Overwrite
Indicates whether to overwrite existing files in the local directory.

.PARAMETER IncludeVersions
Specifies whether to download all versions of the files.

.EXAMPLE
Download files to a local directory:
Download-SharePointFiles -DownloadPath "C:\SharePointDownloads"

.EXAMPLE
Download files and overwrite existing ones:
Download-SharePointFiles -DownloadPath "C:\SharePointDownloads" -Overwrite

.EXAMPLE
Download files including all versions:
Download-SharePointFiles -DownloadPath "C:\SharePointDownloads" -IncludeVersions

.EXAMPLE
Download files, overwrite existing ones, and include all versions:
Download-SharePointFiles -DownloadPath "C:\SharePointDownloads" -Overwrite -IncludeVersions
#>
function Download-SharePointFiles {
        
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $true)]
        [string]$DownloadPath, # The local directory to save files

        [Parameter(Mandatory = $false)]
        [switch]$Overwrite, # Whether to overwrite existing files

        [Parameter(Mandatory = $false)]
        [switch]$IncludeVersions # Whether to download all versions of the files
    )

    # Ensure there is a connection to SharePoint
    $connection = Get-PnPConnection -ErrorAction Stop
        if ($null -ne $connection) {
            Write-Verbose "You are connected to your tenant: $($connection.TenantUrl)"
        } else {
            throw "Please connect and try again"
        }

    # Ensure the download directory exists
    if (-not (Test-Path -Path $DownloadPath)) {
        New-Item -Path $DownloadPath -ItemType Directory | Out-Null
    }

    # Get Context if -IncludeVersions is specified
    if ($IncludeVersions.IsPresent) {
        $Ctx = Get-PnPContext
    }

    # Get all document libraries in the site
    Write-Verbose "Fetching all document libraries in the site..." 
    $Libraries = Get-PnPList | Where-Object { $_.BaseTemplate -eq 101 } # 101 = Document Library

    # Loop through each library and download files
    foreach ($Library in $Libraries) {
        Write-Host "Processing library: $($Library.Title)" -ForegroundColor Yellow

        # Get all files in the library
        $Files = Get-PnPListItem -List $Library.Title -PageSize 1000 -Fields FileLeafRef, FileDirRef | Where-Object { $_.FileSystemObjectType -eq "File" }

        foreach ($File in $Files) {
            $FileUrl = $File["FileRef"]
            $LocalPath = Join-Path $DownloadPath ($File["FileDirRef"] -replace "/", "\") # Convert SharePoint folder structure to local paths
            $FileName = $File["FileLeafRef"]
            $LocalFilePath = Join-Path $LocalPath $FileName

            # Create local directory if it doesn't exist
            if (-not (Test-Path -Path $LocalPath)) {
                New-Item -Path $LocalPath -ItemType Directory | Out-Null
            }

            # Download the current version of the file
            if (-not (Test-Path -Path $LocalFilePath) -or $Overwrite.IsPresent) {
                Write-Host "Downloading file: $FileName" -ForegroundColor Green
                Write-Verbose $FileUrl
                Get-PnPFile -Url $FileUrl  -Path $LocalPath -FileName $FileName -AsFile -Force
            } else {
                Write-Host "File already exists: $FileName. Skipping..." -ForegroundColor Yellow
            }

            # Optionally download all versions
            # got help from https://www.sharepointdiary.com/2018/06/sharepoint-online-download-all-versions-using-powershell.html
            if ($IncludeVersions.IsPresent) {
                Write-Verbose "Fetching versions for: $FileName"
                Write-Verbose $FileUrl
                $pnpfile = Get-PnPFile -Url $FileUrl
                $Versions = Get-PnPProperty -ClientObject $pnpfile -Property Versions

                if ($Versions.Count -gt 0) {
                    foreach ($Version in $Versions) {
                        # Construct version filename
                        $VersionFileName = "$($LocalPath)\$($FileName)_$($Version.VersionLabel)"
          
                        #Get Contents of the File Version
                        $VersionStream = $Version.OpenBinaryStream()
                        $Ctx.ExecuteQuery()
                
                        #Download File version to local disk
                        [System.IO.FileStream] $FileStream = [System.IO.File]::Open($VersionFileName,[System.IO.FileMode]::OpenOrCreate)
                        $VersionStream.Value.CopyTo($FileStream)
                        $FileStream.Close()
                        
                        Write-Host -f Green "Downloading Version $($Version.VersionLabel) to:" $VersionFileName
                        
                    }
                } else {
                    if ($PSCmdlet.MyInvocation.BoundParameters["Verbose"]) {
                        Write-Host "No versions available for file: $FileName" -ForegroundColor Yellow
                    } 

                    
                }
            }
        }
    }

    Write-Host "Download completed!" -ForegroundColor Cyan
}


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***


## Source Credit

Sample first appeared on [https://pnp.github.io/cli-microsoft365/sample-scripts/spo/spo-download-all-doclibs/](https://pnp.github.io/cli-microsoft365/sample-scripts/spo/spo-download-all-doclibs/)

## Contributors

| Author(s) |
|-----------|
| Todd Klindt (https://www.toddklindt.com/blog) |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-download-all-doclibs" aria-hidden="true" />



# Trim file versions and archive Site Collection using Microsoft365 Archive

## Summary

As of time of writing, the Microsoft 365 Archive is just out of preview. The current out of the box way to archvive a site is to make the Site Collection read-only, but I would expect that the Microsoft 365 Archive will be the way to go in the future, at least when the feature has been enabled on your tenant :-)
In the meantime we have to do the archiving the proper way ourself and this script will trim file versions and then archive the site collection to the Microsoft 365 Archive.
![Example Screenshot](assets/example.png)


# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
function Invoke-SpoArchiveSite {
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$SiteUrl,

        [Parameter(Mandatory = $true)]
        [ValidateRange(1, 500)]
        [int]$VersionsToKeep,

        [Parameter(Mandatory = $false)]
        [ValidateNotNullOrEmpty()]
        [string]$LibraryUrl,

        [Parameter(Mandatory = $false, HelpMessage = "Optional path to store the trimming report as CSV.")]
        [ValidateNotNullOrEmpty()]
        [string]$ReportPath,

        [Parameter(Mandatory = $false)]
        [switch]$Force
    )

    begin {
        Write-Host "Ensuring CLI session is authenticated..."
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to authenticate with CLI for Microsoft 365. Details: $loginOutput"
        }

        $Summary = [pscustomobject]@{
            FilesEvaluated = 0
            VersionsTrimmed = 0
            VersionRemovalFailures = 0
            FileEnumerationFailures = 0
            ArchiveAttempted = $false
            ReportPath = $null
        }

        if (-not $ReportPath) {
            $timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
            $ReportPath = Join-Path -Path (Get-Location) -ChildPath "spo-archive-report-$timestamp.csv"
        }

        $Summary.ReportPath = $ReportPath
        $ReportRows = [System.Collections.Generic.List[psobject]]::new()

        if ($LibraryUrl) {
            Write-Host "Targeting library: $LibraryUrl" -ForegroundColor Cyan
        } else {
            Write-Host "Processing all document libraries in the site." -ForegroundColor Cyan
        }
    }

    process {
        $listsJson = if ($LibraryUrl) {
            m365 spo list list --webUrl $SiteUrl --output json --filter "RootFolder.ServerRelativeUrl eq '$LibraryUrl'" 2>&1
        } else {
            m365 spo list list --webUrl $SiteUrl --output json --query "[?BaseTemplate==`101` || BaseTemplate==`700`]" 2>&1
        }
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to enumerate document libraries. CLI output: $listsJson"
        }

        $lists = $listsJson | ConvertFrom-Json
        if (-not $lists) {
            Write-Warning "No document libraries matched the provided criteria."
            return
        }

        foreach ($list in $lists) {
            Write-Host "Inspecting library '$($list.Title)'..." -ForegroundColor Yellow

            $filesJson = m365 spo file list --webUrl $SiteUrl --folderUrl $list.RootFolder.ServerRelativeUrl --recursive --fields "UniqueId,Name,ServerRelativeUrl" --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to list files in '$($list.Title)'. CLI output: $filesJson"
                $Summary.FileEnumerationFailures++
                continue
            }

            $files = $filesJson | ConvertFrom-Json
            if (-not $files) {
                Write-Host "No files found in '$($list.Title)'." -ForegroundColor DarkGray
                continue
            }

            foreach ($file in $files) {
                $Summary.FilesEvaluated++

                $versionsJson = m365 spo file version list --webUrl $SiteUrl --fileId $file.UniqueId --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Could not retrieve versions for '$($file.ServerRelativeUrl)'. CLI output: $versionsJson"
                    $Summary.VersionRemovalFailures++
                    continue
                }

                $versions = $versionsJson | ConvertFrom-Json
                if ($versions.Count -le $VersionsToKeep) {
                    continue
                }

                $versionsToRemove = $versions | Sort-Object -Property Created -Descending | Select-Object -Skip $VersionsToKeep
                $removedCount = 0
                $failedRemovalCount = 0

                foreach ($version in $versionsToRemove) {
                    if ($PSCmdlet.ShouldProcess($file.ServerRelativeUrl, "Remove version $($version.VersionLabel)")) {
                        $removeArgs = @(
                            'spo', 'file', 'version', 'remove',
                            '--webUrl', $SiteUrl,
                            '--fileId', $file.UniqueId,
                            '--label', $version.VersionLabel,
                            '--output', 'json'
                        )

                        if ($Force) {
                            $removeArgs += '--force'
                        }

                        $removeOutput = m365 @removeArgs 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to remove version '$($version.VersionLabel)' for '$($file.ServerRelativeUrl)'. CLI: $removeOutput"
                            $Summary.VersionRemovalFailures++
                            $failedRemovalCount++
                            continue
                        }
                        $Summary.VersionsTrimmed++
                        $removedCount++
                    }
                }

                if ($versionsToRemove.Count -gt 0) {
                    $ReportRows.Add([pscustomobject]@{
                        SiteUrl = $SiteUrl
                        LibraryTitle = $list.Title
                        FileUrl = $file.ServerRelativeUrl
                        VersionsBefore = $versions.Count
                        VersionsAttempted = $versionsToRemove.Count
                        VersionsRemoved = $removedCount
                        VersionsFailed = $failedRemovalCount
                        VersionsRemaining = $versions.Count - $removedCount
                    }) | Out-Null
                }
            }
        }

        if ($PSCmdlet.ShouldProcess($SiteUrl, 'Archive site collection')) {
            $archiveArgs = @(
                'spo', 'site', 'archive',
                '--url', $SiteUrl,
                '--output', 'json'
            )

            if ($Force) {
                $archiveArgs += '--force'
            }

            $archiveOutput = m365 @archiveArgs 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to archive site '$SiteUrl'. CLI output: $archiveOutput"
            }

            $Summary.ArchiveAttempted = $true
            Write-Host "Archive request submitted for '$SiteUrl'." -ForegroundColor Green
        }
    }

    end {
        Write-Host "Summary:" -ForegroundColor Cyan
        Write-Host "  Files evaluated: $($Summary.FilesEvaluated)"
        Write-Host "  Versions trimmed: $($Summary.VersionsTrimmed)"
        Write-Host "  Version failures: $($Summary.VersionRemovalFailures)"
        Write-Host "  Library enumeration failures: $($Summary.FileEnumerationFailures)"
        Write-Host "  Archive requested: $($Summary.ArchiveAttempted)"

        if ($ReportRows.Count -gt 0) {
            $reportDirectory = Split-Path -Path $Summary.ReportPath -Parent
            if ([string]::IsNullOrWhiteSpace($reportDirectory)) {
                $reportDirectory = (Get-Location).Path
            }

            if (-not (Test-Path -Path $reportDirectory)) {
                try {
                    New-Item -ItemType Directory -Path $reportDirectory -Force | Out-Null
                }
                catch {
                    Write-Warning "Unable to create report directory '$reportDirectory'. $_"
                }
            }

            try {
                $ReportRows | Export-Csv -Path $Summary.ReportPath -NoTypeInformation -Encoding UTF8
                Write-Host "Report saved to $($Summary.ReportPath)" -ForegroundColor Cyan
            }
            catch {
                Write-Warning "Failed to write report file '$($Summary.ReportPath)'. $_"
            }
        }
        else {
            Write-Host "No version changes detected; report not generated." -ForegroundColor DarkGray
        }
    }
}

# Example usage
Invoke-SpoArchiveSite -SiteUrl "https://contoso.sharepoint.com/sites/projects" -VersionsToKeep 5 -WhatIf
```

# [PnP PowerShell](#tab/pnpps)

```powershell

#sample showing how to trim file versions and archive a SharePoint Site collection

$siteUrl = "https://contoso.sharepoint.com/sites/contoso"
#use -interactive when working locally, managed identity when running in Azure and I guess you have to use a certificate when dealing with SAAS
$conn = Connect-PnPOnline -Url $siteUrl -Interactive -ReturnConnection

function DeleteVersions($siteUrl, $ListName, $listitemID, $versionsToKeep)
{
    try 
    {
        #get list of all lists in this site
        if($ListName)
        {
            $list = Get-PnPList -Identity $ListName -Connection $conn -ErrorAction SilentlyContinue
            if($list)
            {
                $listitems = Get-PnPListItem -List $list -Connection $conn
                if($listitemID)
                {
                    $listitem = $listitems | Where-Object {$_.ID -eq $listitemID}
                    if($listitem)
                    {
                        $file = Get-PnPFile  -Url $listitem["FileRef"] -AsFileObject -ErrorAction SilentlyContinue -Connection $conn           
                        if($file)
                        {
                            $fileversions = Get-PnPFileVersion -Url $listitem["FileRef"] -Connection $conn
                            if($fileversions)
                            {
                                if($fileversions.Count -gt $versionsToKeep)
                                {
                                    $DeleteVersionList = ($fileversions[0..$($fileversions.Count - $versionsToKeep)])
                                    $element = "" | Select-Object SiteUrl, siteName, ListTitle, itemName, fileType, Modified, versioncount, FileSize
                                    $element.SiteUrl = $siteUrl
                                    $element.siteName = $conn.Name
                                    $element.ListTitle = $list.Title
                                    $element.itemName = $file.Name
                                    $fileextention = $item["FileLeafRef"].Substring($item["FileLeafRef"].LastIndexOf(".")+1)
                                    $element.fileType = $fileextention
                                    $element.Modified = $file.TimeLastModified.tostring()
                                    $element.versioncount = $fileversions.Count
                                    $element.fileSize = $file.Length
                                    
                                    $arraylist.Add($element) | Out-Null                        
                                    
                                    foreach($VersionToDelete in $DeleteVersionList) 
                                    {
                                        Remove-PnPFileVersion -Url $listitem["FileRef"] -Identity $VersionToDelete.Id –Force -Connection $conn            
                                    }
                                }
                                else {
                                    write-host "no versions to delete"
                                }
                            }                            
                        }
                        else {
                            write-host "file not found" -ForegroundColor Red
                        }
                    }
                }
                else 
                {
                    foreach($listitem in $listitems)
                    {
                        $file = Get-PnPFile  -Url $listitem["FileRef"] -AsFileObject -ErrorAction SilentlyContinue -Connection $conn           
                        if($file)
                        {
                            $fileversions = Get-PnPFileVersion -Url $listitem["FileRef"] -Connection $conn
                            if($fileversions)
                            {
                                Write-Host "fileversions found $($fileversions.Count)"
                                if($fileversions.Count -gt $versionsToKeep)
                                {
                                    $number =$fileversions.Count - $versionsToKeep
                                    $DeleteVersionList = ($fileversions[0..$number])
                                    $element = "" | Select-Object SiteUrl, ListTitle, itemName, fileType, Modified, versioncount, FileSize
                                    $element.SiteUrl = $siteUrl
                                    $element.ListTitle = $list.Title
                                    $element.itemName = $file.Name
                                    $fileextention = $listitem["FileLeafRef"].Substring($listitem["FileLeafRef"].LastIndexOf(".")+1)
                                    $element.fileType = $fileextention
                                    $element.Modified = $file.TimeLastModified.tostring()
                                    $element.versioncount = $fileversions.Count
                                    $element.fileSize = $file.Length
                                    
                                    $arraylist.Add($element) | Out-Null
                                    foreach($VersionToDelete in $DeleteVersionList) 
                                    {
                                        Remove-PnPFileVersion -Url $listitem["FileRef"] -Identity $VersionToDelete.Id –Force -Connection $conn            
                                    }
                                }
                                else {
                                    write-host "no versions to delete"
                                }                                
                            }
                            else {
                                write-host "fileversions not found" -ForegroundColor Yellow
                            }                            
                        }
                        else {
                            write-host "file not found" -ForegroundColor Yellow
                        }
                    }
                }
            }            
        }
        else 
        {
            # you can trim all lists in a site here
            $lists = Get-PnPList -Identity $ListName -Connection $conn -ErrorAction SilentlyContinue | Where-Object { $_.Hidden -eq $false -and $_.BaseType -eq "DocumentLibrary" }
            foreach($list in $lists)
            {
                DeleteVersions -siteUrl $siteUrl -ListName $list.Title -versionsToKeep $versionsToKeep -Connection $conn
            }
        }            
    }
    catch 
    {
        Write-Output "Ups an exception was thrown : $($_.Exception.Message)" -ForegroundColor Red
    }  
}

#trim file versions
DeleteVersions -siteUrl $siteUrl -versionsToKeep 5 -Connection $conn #trim all libraries to 5 versions
#archive the site collection
Set-PnPSiteArchiveState $siteUrl -ArchiveState Archived -Connection $conn


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-trim-and-m365-archive-sitecollection" aria-hidden="true" />

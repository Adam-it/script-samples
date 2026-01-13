

# Get Checked Out Files including those with no checked in version

## Summary

There are scenarios when files uploaded won't have **checked-in version** which will make the files visible only to their uploader. Two possible scenarios.

1. When **Require Check Out** option under versioning settings of any library is set to "Yes" and the uploader forget to check in the file. 

2. When there are required fields and end user uses OneDrive to save a newly created office file.

![PnP Powershell result](assets/preview.png)

Files which have no checked in versions have the following issues

- Invisibility: They evade search results, remaining hidden from intended audiences.
- Backup Issues: These files are not backed up, risking data loss.
- Mass update of those files failed

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(HelpMessage = "Optional OData filter for sites (e.g., \"Url -like 'project'\" or \"Template eq 'TEAMCHANNEL#1'\").")]
    [string]$SiteFilter,

    [Parameter(HelpMessage = "Include OneDrive sites in the audit.")]
    [switch]$IncludeOneDrive,

    [Parameter(HelpMessage = "Output directory for CSV report. Defaults to current location.")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $ErrorActionPreference = 'Stop'

    Write-Host "Ensuring CLI for Microsoft 365 login status..." -ForegroundColor Cyan
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure login to CLI for Microsoft 365. Please run 'm365 login' first."
    }

    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath -PathType Container)) {
            throw "Output path '$OutputPath' does not exist or is not a directory."
        }
    }

    $ExcludedLibraries = @(
        "Converted Forms", "Master Page Gallery", "Customized Reports", "Form Templates",
        "List Template Gallery", "Theme Gallery", "Apps for SharePoint", "Reporting Templates",
        "Solution Gallery", "Style Library", "Web Part Gallery", "Site Assets", "wfpub",
        "Site Pages", "Images", "MicroFeed", "Pages", "Preservation Hold Library",
        "Site Collection Documents", "Site Collection Images", "AppPages", "Apps for Office"
    )

    $script:CheckedOutFiles = @()
    $script:Summary = @{
        SitesProcessed     = 0
        LibrariesProcessed = 0
        FilesCheckedOut    = 0
        FilesNoVersion     = 0
        Failures           = 0
    }

    $timestamp = Get-Date -Format "yyyy-MM-dd-HH-mm"
    $transcriptPath = Join-Path -Path $OutputPath -ChildPath "CheckedOutFiles-Transcript-$timestamp.log"
    Start-Transcript -Path $transcriptPath
}

process {
    try {
        Write-Host "`nRetrieving sites from tenant..." -ForegroundColor Cyan
        
        # Build command with conditional options (no Invoke-Expression)
        if ($SiteFilter -and $IncludeOneDrive) {
            Write-Verbose "Executing: m365 spo site list --filter '$SiteFilter' --withOneDriveSites"
            $sitesJson = m365 spo site list --filter $SiteFilter --withOneDriveSites --output json
        }
        elseif ($SiteFilter) {
            Write-Verbose "Executing: m365 spo site list --filter '$SiteFilter'"
            $sitesJson = m365 spo site list --filter $SiteFilter --output json
        }
        elseif ($IncludeOneDrive) {
            Write-Verbose "Executing: m365 spo site list --withOneDriveSites"
            $sitesJson = m365 spo site list --withOneDriveSites --output json
        }
        else {
            Write-Verbose "Executing: m365 spo site list"
            $sitesJson = m365 spo site list --output json
        }
        
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sites. CLI: $sitesJson"
        }

        $sites = @($sitesJson | ConvertFrom-Json)
        Write-Host "Found $($sites.Count) site(s) to process." -ForegroundColor White

        foreach ($site in $sites) {
            $script:Summary.SitesProcessed++
            Write-Host "`nProcessing site: $($site.Url)" -ForegroundColor Magenta

            try {
                Write-Verbose "Executing: m365 spo list list --webUrl $($site.Url) --filter"
                $listsJson = m365 spo list list --webUrl $site.Url --filter "BaseTemplate eq 101 and Hidden eq false" --output json
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve lists for site '$($site.Url)'. CLI: $listsJson"
                    $script:Summary.Failures++
                    continue
                }

                $lists = @($listsJson | ConvertFrom-Json)
                $filteredLists = $lists | Where-Object { $_.Title -notin $ExcludedLibraries }

                if ($filteredLists.Count -eq 0) {
                    Write-Verbose "No document libraries found in site '$($site.Url)' (after filtering)."
                    continue
                }

                Write-Host "  Found $($filteredLists.Count) document library(ies)." -ForegroundColor White

                foreach ($list in $filteredLists) {
                    $script:Summary.LibrariesProcessed++
                    Write-Host "    Processing library: $($list.Title)" -ForegroundColor Yellow

                    try {
                        Write-Verbose "Executing: m365 spo listitem list --webUrl $($site.Url) --listId $($list.Id) --fields"
                        $itemsJson = m365 spo listitem list --webUrl $site.Url --listId $list.Id --fields "FileLeafRef,FileRef,CheckoutUser/Title,CheckoutUser/Id,_UIVersionString,FSObjType" --filter "FSObjType eq 0 and CheckoutUser ne null" --output json
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to retrieve items for library '$($list.Title)' in site '$($site.Url)'. CLI: $itemsJson"
                            $script:Summary.Failures++
                            continue
                        }

                        $items = @($itemsJson | ConvertFrom-Json)

                        if ($items.Count -eq 0) {
                            Write-Verbose "No checked out files found in library '$($list.Title)'."
                            continue
                        }

                        Write-Host "      Found $($items.Count) checked out file(s)." -ForegroundColor Green

                        foreach ($item in $items) {
                            $script:Summary.FilesCheckedOut++

                            $uiVersionString = $item._UIVersionString
                            $noCheckedInVersion = "No"

                            if ($uiVersionString -match '^0\.') {
                                $noCheckedInVersion = "Yes"
                                $script:Summary.FilesNoVersion++
                            }

                            $checkedOutByUser = if ($item.CheckoutUser) { $item.CheckoutUser.Title } else { "Unknown" }

                            $script:CheckedOutFiles += [PSCustomObject]@{
                                SiteUrl            = $site.Url
                                FileUrl            = $item.FileRef
                                CheckedOutBy       = $checkedOutByUser
                                NoCheckedInVersion = $noCheckedInVersion
                                VersionLabel       = $uiVersionString
                            }
                        }
                    }
                    catch {
                        Write-Warning "Error processing library '$($list.Title)' in site '$($site.Url)': $($_.Exception.Message)"
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
        Write-Error "Critical error during processing: $($_.Exception.Message)"
        throw
    }
}

end {
    if ($script:CheckedOutFiles.Count -gt 0) {
        $csvFileName = "CheckedOutFiles-$timestamp.csv"
        $csvPath = Join-Path -Path $OutputPath -ChildPath $csvFileName
        $script:CheckedOutFiles | Export-Csv -Path $csvPath -NoTypeInformation -Force
        Write-Host "`nCSV report exported to: $csvPath" -ForegroundColor Green
    }
    else {
        Write-Host "`nNo checked out files found across all sites." -ForegroundColor Yellow
    }

    $failureColor = if ($script:Summary.Failures -gt 0) { "Red" } else { "Green" }

    Write-Host "`n=== Summary ===" -ForegroundColor Cyan
    Write-Host "Sites Processed        : $($script:Summary.SitesProcessed)" -ForegroundColor White
    Write-Host "Libraries Processed    : $($script:Summary.LibrariesProcessed)" -ForegroundColor White
    Write-Host "Files Checked Out      : $($script:Summary.FilesCheckedOut)" -ForegroundColor White
    Write-Host "Files with No Version  : $($script:Summary.FilesNoVersion)" -ForegroundColor Yellow
    Write-Host "Failures               : $($script:Summary.Failures)" -ForegroundColor $failureColor

    Stop-Transcript
}

# Usage examples:
#
# Example 1: Audit all SharePoint sites (excluding OneDrive)
# .\Get-CheckedOutFiles.ps1
#
# Example 2: Audit only sites with 'project' in the URL
# .\Get-CheckedOutFiles.ps1 -SiteFilter "Url -like 'project'"
#
# Example 3: Include OneDrive sites with verbose logging
# .\Get-CheckedOutFiles.ps1 -IncludeOneDrive -Verbose
#
# Example 4: Audit Team Channel sites with custom output path
# .\Get-CheckedOutFiles.ps1 -SiteFilter "Template eq 'TEAMCHANNEL#1'" -OutputPath "C:\Reports"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell
#Set Parameters
$AdminCenterURL="https://contoso-admin.sharepoint.com/"
Connect-PnPOnline -Url $AdminCenterURL -Interactive
$dateTime = (Get-Date).toString("dd-MM-yyyy")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "checkedoutfiles-" + $dateTime + ".csv"
$OutPutView = $directorypath + "\Logs\"+ $fileName
# Array to Hold Result - PSObjects
$filesCollection = @()
 
#Array to Skip System Lists and Libraries
$SystemLists = @("Converted Forms", "Master Page Gallery", "Customized Reports", "Form Templates", "List Template Gallery", "Theme Gallery","Apps for SharePoint",
                            "Reporting Templates", "Solution Gallery", "Style Library", "Web Part Gallery","Site Assets", "wfpub", "Site Pages", "Images", "MicroFeed","Pages")
 
$m365Sites = Get-PnPTenantSite -Detailed | Where-Object {($_.Url -like '*/intranet-*' -or  $_.Url -like '*/team-*' -or $_.Template -eq 'TEAMCHANNEL#1') -and $_.Template -ne 'RedirectSite#0' }
$m365Sites | ForEach-Object {
$siteUrl = $_.Url;    
Connect-PnPOnline -Url $siteUrl -Interactive
 
$Ctx = Get-PnPContext
#Get the List
write-host  $siteUrl  
 Get-PnPList  | Where {$_.Hidden -eq $false -and $SystemLists -notcontains $_.Title -and $_.BaseTemplate -eq 101 } | ForEach-Object {
#Get All Checked-Out Files
$CheckedOutFiles = $_.GetCheckedOutFiles()
$Ctx.Load($CheckedOutFiles)
$Ctx.ExecuteQuery()
#Check-in All Files Checked out to the User
$CheckedOutFiles | ForEach-Object {

        $user = (Get-PnPUser -Identity $_.CheckedoutById -ErrorAction Ignore) ?? $_.CheckedoutById
        $ExportVw = New-Object PSObject
        $ExportVw | Add-Member -MemberType NoteProperty -name "Site URL" -value $siteUrl
        $ExportVw | Add-Member -MemberType NoteProperty -name "File Url" -value $_.ServerRelativePath.DecodedUrl
        $ExportVw | Add-Member -MemberType NoteProperty -name "Checked Out By" -value $user.Title
        $ExportVw | Add-Member -MemberType NoteProperty -name "No Checked in version" -value "Yes"
        $filesCollection += $ExportVw
    }
 
$alldocs = (Get-PnPListItem -List $_  -PageSize 1000 | where-object{ $null -ne $_.FieldValues.CheckoutUser} )
 
$alldocs | ForEach-Object {
        $ExportVw = New-Object PSObject
        $ExportVw | Add-Member -MemberType NoteProperty -name "Site URL" -value $siteUrl
        $ExportVw | Add-Member -MemberType NoteProperty -name "File Url" -value $_.FieldValues.FileRef
        $ExportVw | Add-Member -MemberType NoteProperty -name "Checked Out By" -value $_.FieldValues.CheckoutUser.LookupValue
        $ExportVw | Add-Member -MemberType NoteProperty -name "No Checked in version" -value "No"
        $filesCollection += $ExportVw
  }
 }
}
# Export the result array to CSV file
$filesCollection | sort-object "File Url" |Export-CSV $OutPutView -Force -NoTypeInformation
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Discovering All Checked Out Files including those with no checked in versions with PnP PowerShell](https://reshmeeauckloo.com/posts/powershell_getallfileswithnocheckedinversion/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011)|
| [Adam Wójcik](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-checkedoutfiles-nocheckedinversion" aria-hidden="true" />



# Get all files in a Document Library along with Created By and Modified By

## Summary

A customer recently wanted to find out who the most active users were in each site. They were planning for a migration and they wanted an idea who was creating and modifying the most files so they could bring them into the migration planning and testing. This script walks through the site's document libraries, lists each file, when it was created and by whom, and when it was last modified by whom. It exports this to a CSV file. The customer can bring this CSV file into Excel and slice the data to the their heart's content. 

![Example Screenshot](assets/example.png)

For this customer they looped through a list of sites gotten from Get-PnPTenantSite and ran this code in a ForEach block. The Export-CSV command uses -Append, so all of the results were stored in one large file.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site to inventory")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)')]
    [string]$SiteUrl,
    
    [Parameter(HelpMessage = "Path where the CSV report will be saved")]
    [string]$OutputPath = (Get-Location).Path,
    
    [Parameter(HelpMessage = "Include Site Pages library in the inventory")]
    [switch]$IncludeSitePages
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $logFile = Join-Path $OutputPath "FilesAndCreators_Log_$timestamp.txt"
    Start-Transcript -Path $logFile

    Write-Host "Starting file inventory for $SiteUrl..." -ForegroundColor Cyan

    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath)) {
            throw "Output path '$OutputPath' does not exist. Please provide a valid path."
        }
    }

    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure login to Microsoft 365. Please run 'm365 login' first."
    }
    Write-Verbose "Login verified successfully."

    $script:FileCollection = @()
    $script:Summary = @{
        Libraries = 0
        Files = 0
        Failures = 0
    }
}

process {
    Write-Verbose "Retrieving document libraries from site..."
    $listJson = m365 spo list list --webUrl $SiteUrl --filter "BaseTemplate eq 101 and Hidden eq false" --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve libraries. CLI: $listJson"
        return
    }

    $libraries = @($listJson | ConvertFrom-Json)
    
    if (-not $IncludeSitePages) {
        $libraries = $libraries | Where-Object { $_.Title -ne "Site Pages" }
    }

    Write-Host "Found $($libraries.Count) document libraries to process." -ForegroundColor Green
    $script:Summary.Libraries = $libraries.Count

    foreach ($library in $libraries) {
        Write-Verbose "Processing library: $($library.Title)"
        
        try {
            $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listId $library.Id --fields "FileRef,Created,Modified,Author/Title,Editor/Title,FSObjType" --filter "FSObjType eq 0" --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve items from library '$($library.Title)'. CLI: $itemsJson"
                $script:Summary.Failures++
                continue
            }

            $items = @($itemsJson | ConvertFrom-Json)
            Write-Verbose "Found $($items.Count) files in '$($library.Title)'"
            $script:Summary.Files += $items.Count

            foreach ($item in $items) {
                $createdBy = if ($item.Author -and $item.Author.Title) {
                    $item.Author.Title
                }
                elseif ($item.AuthorId) {
                    "User ID: $($item.AuthorId)"
                }
                else {
                    "Unknown"
                }
                
                $modifiedBy = if ($item.Editor -and $item.Editor.Title) {
                    $item.Editor.Title
                }
                elseif ($item.EditorId) {
                    "User ID: $($item.EditorId)"
                }
                else {
                    "Unknown"
                }

                $script:FileCollection += [PSCustomObject]@{
                    Library = $library.Title
                    FileRef = $item.FileRef
                    Created_By = $createdBy
                    Created = $item.Created
                    Modified_By = $modifiedBy
                    Modified = $item.Modified
                }
            }
        }
        catch {
            Write-Warning "Error processing library '$($library.Title)': $($_.Exception.Message)"
            $script:Summary.Failures++
        }
    }
}

end {
    if ($script:FileCollection.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "FilesAndCreators_$timestamp.csv"
        $script:FileCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "`nCSV report exported to: $csvPath" -ForegroundColor Green
    }
    else {
        Write-Host "`nNo files found to export." -ForegroundColor Yellow
    }

    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Libraries Processed: $($script:Summary.Libraries)" -ForegroundColor White
    Write-Host "Total Files Found: $($script:Summary.Files)" -ForegroundColor White
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failed Libraries: $($script:Summary.Failures)" -ForegroundColor Red
    }
    else {
        Write-Host "Failed Libraries: 0" -ForegroundColor Green
    }
    Write-Host "============================="-ForegroundColor Cyan

    Stop-Transcript
}

# Usage examples:
#
# Example 1: Basic usage with mandatory site URL
# .\Get-FilesAndCreatorsModifiers.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x"
#
# Example 2: Include Site Pages library
# .\Get-FilesAndCreatorsModifiers.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -IncludeSitePages
#
# Example 3: Custom output path with verbose logging
# .\Get-FilesAndCreatorsModifiers.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -OutputPath "C:\Reports" -Verbose

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

# [PnP PowerShell](#tab/pnpps)

```powershell

# Connect to the site we want the inventory from
Connect-PnPOnline -Url https://contoso.sharepoint.com -Interactive

# Get all of the Libraries we want
$LibraryList = Get-PnPList -Includes IsSystemList,RootFolder | Where-Object {($_.BaseType -eq "DocumentLibrary" -and $_.IsSystemList -eq $False) -or ($_.Title -eq "Site Pages")}

# All the files in all the document libraries
$LibraryList | ForEach-Object{Get-PnPListItem -List $_.RootFolder.Name | Where-Object{$_.FieldValues.FSObjType -ne 1} | Select-Object @{n="FileRef";e={$_.FieldValues.FileRef}},@{n="Created_x0020_By";e={$($_.FieldValues.Created_x0020_By).split("|")[2]}},@{n="Created";e={$_.FieldValues.Created}},@{n="Modified_x0020_By";e={$($_.FieldValues.Modified_x0020_By).split("|")[2]}},@{n="Modified";e={$_.FieldValues.Modified}}} | Export-Csv -Path .\FilesAndOwners.csv -Append

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Todd Klindt](https://www.toddklindt.com)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-files-and-creators-modifiers" aria-hidden="true" />

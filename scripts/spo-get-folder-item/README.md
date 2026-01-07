

# Get Folder Item properties

## Summary

This script retrieves file properties from large libraries, especially within specific folders, along with their associated properties. An alternative to **Get-PnPFolderItem** which may not work efficiently with large libraries.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory, HelpMessage = "Title of the library (e.g., 'Shared Documents')")]
    [string]$LibraryName,
    
    [Parameter(Mandatory, HelpMessage = "Folder path pattern (e.g., '/sites/test/Shared Documents/folder' or '*Shared Documents/folder*')")]
    [string]$FolderPath,
    
    [Parameter(HelpMessage = "Name of the custom field to extract unique values from (e.g., 'PPF_Comments', 'Issue_Comments')")]
    [string]$CustomFieldName = "PPF_Comments",
    
    [Parameter(HelpMessage = "Path where the CSV report will be saved")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath)) {
            throw "Output path does not exist: $OutputPath"
        }
    }
    
    $transcriptPath = Join-Path $OutputPath "FolderItems_$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to Microsoft 365"
    }
    
    $script:UniqueValues = @()
    $script:Summary = @{
        ItemsFound = 0
        UniqueCategories = 0
    }
}

process {
    try {
        Write-Verbose "Retrieving library: $LibraryName"
        $listJson = m365 spo list get --webUrl $SiteUrl --title $LibraryName --properties "Id,Title,ItemCount" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to get library '$LibraryName'. Error: $listJson"
        }
        $list = @($listJson | ConvertFrom-Json)[0]
        
        $folderPattern = $FolderPath -replace '^\\*', '' -replace '\\*$', ''
        
        Write-Host "Searching for files in folder: $folderPattern" -ForegroundColor Cyan
        Write-Verbose "Total items in library: $($list.ItemCount)"
        
        $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listId $list.Id --fields "FileRef,FileLeafRef,FSObjType,$CustomFieldName" --filter "FSObjType eq 0" --query "[?contains(FileRef, '$folderPattern')]" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to get list items. Error: $itemsJson"
        }
        $filteredItems = @($itemsJson | ConvertFrom-Json)
        
        Write-Host "Found $($filteredItems.Count) file(s) matching folder pattern" -ForegroundColor Green
        $script:Summary.ItemsFound = $filteredItems.Count
        
        $uniqueCategories = [System.Collections.ArrayList]@()
        
        foreach ($item in $filteredItems) {
            if ($item.$CustomFieldName -and $uniqueCategories.Name -notcontains $item.$CustomFieldName) {
                $uniqueCategories.Add([PSCustomObject]@{
                    Name = $item.$CustomFieldName
                }) | Out-Null
                Write-Verbose "Found unique value: $($item.$CustomFieldName)"
            }
        }
        
        $script:UniqueValues = $uniqueCategories
        $script:Summary.UniqueCategories = $uniqueCategories.Count
    }
    catch {
        Write-Warning "Error processing library: $_"
        throw
    }
}

end {
    $csvPath = Join-Path $OutputPath "categories.csv"
    $script:UniqueValues | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8 -Delimiter "|"
    
    Write-Host "`n=== Script Summary ===" -ForegroundColor Cyan
    Write-Host "Files Found: $($script:Summary.ItemsFound)" -ForegroundColor White
    Write-Host "Unique Categories: $($script:Summary.UniqueCategories)" -ForegroundColor Green
    Write-Host "CSV Exported: $csvPath" -ForegroundColor White
    Write-Host "Transcript: $transcriptPath" -ForegroundColor White
    
    Stop-Transcript
}

# Usage examples:
#
# Example 1: Extract unique PPF_Comments from folder
# .\Get-FolderItemProperties.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/test" -LibraryName "Shared Documents" -FolderPath "/sites/test/Shared Documents/folder"
#
# Example 2: Use wildcard pattern with custom field
# .\Get-FolderItemProperties.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/test" -LibraryName "Documents" -FolderPath "*Documents/subfolder*" -CustomFieldName "Issue_Comments"
#
# Example 3: Custom output path
# .\Get-FolderItemProperties.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/test" -LibraryName "Documents" -FolderPath "*Documents/folder*" -OutputPath "C:\Reports"
#
# Example 4: Verbose output
# .\Get-FolderItemProperties.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/test" -LibraryName "Documents" -FolderPath "*Documents/folder*" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```PowerShell
$SiteUrl = Read-Host -Prompt "Enter site collection URL "; #e.g "https://contoso.sharepoint.com/sites/test"
Connect-PnPOnline -url $SiteUrl -Interactive
$listName = Read-Host -Prompt "Enter the library name, e.g. 'Shared Documents'" 
$FolderSiteRelativeURL = Read-Host -Prompt "Enter relative folder url starting with *, e.g. '*Shared Documents/folder' "; #e.g."*Shared Documents/folder/subfolder-folder/subfolder-subfolder-folder*"

$list = Get-PnPList $listName
$global:counter = 0
#Retrieving all items within the folder which is not a folder
$items = Get-PnPListItem -List $listName -PageSize 500 -Fields FileLeafRef,FileRef,PPF_Comments -ScriptBlock `
      { Param($items) $global:counter += $items.Count; Write-Progress -PercentComplete `
    ($global:Counter / ($List.ItemCount) * 100) -Activity "Getting folders from List:" -Status "Processing Items $global:Counter to $($List.ItemCount)";} `
    | Where {$_.FileSystemObjectType -ne "Folder" -and $_.FieldValues.FileRef -like $FolderSiteRelativeURL}
 
$type = [System.Collections.ArrayList]@();
 
$items | foreach-object {
    if($_.FieldValues.Issue_Comments){
        if($type -notcontains $_.FieldValues.Issue_Comments){
            $type.Add([PSCustomObject]@{
                Name = $_.FieldValues.Issue_Comments
            });
            write-host $_.FieldValues.Issue_Comments;
        }
   }
}
 
$type | Export-Csv -Path "C:\temp\categories.csv" -NoTypeInformation -Force -Delimiter "|"
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***
## Source Credit

Sample first appeared on [Pnp Powershell Get Folder Item](https://reshmeeauckloo.com/posts/pnp-powershell-get-folder-item/)

## Contributors
| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Reshmee Auckloo (script)](https://github.com/reshmee011)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-folder-item" aria-hidden="true" />



# Copies the structure of a directory to a SharePoint list

## Summary

Say we are trying to migrate file shares to SharePoint. We want to get all the individual directories into a SharePoint list so that we can somehow trigger a migration using the SharePoint Migration Manager (https://tenant-admin.SharePoint.com/_layouts/15/online/AdminHome.aspx#/migration/fileshare).

This script will scan a selected file share for all folders and create an entry in a SharePoint list for each folder found. This list then can be used to trigger migrations on specific folders to specific SharePoint location in a number of different ways (PowerApps, PowerAutomate, SPFX, export to excel).

Before running the script we must first create a SharePoint list to hold the folder structure. I created an empty list and renamed the Title column to FileSharePath.
I then added 20 text columns (my directory is 20 levels deep) name Level1...Level20 and added an index to each (so we can filter on them easily). I also added a number column called Level that represents the depth in the hierarchy and 3 additional columns (SharePointSite, DocLibrary and DocSubfolder) that a user can enter into the list online to facilitate migrations using the SharePoint Migration Service.

The final list is shown below:
![Example Screenshot](assets/LISTSTRUCTURE.PNG)

After running the script the list will be populated with one row for each folder in your
fileshare.

So now you can go to the list and enter the url for a SharePoint site, a document library and an optional subfolder for each directory. 

Note: The longest path in the directory structure cannot exceed 260 characters!

There are multiple ways to trigger a migration with this info. The simplest is to create a view with just the FileSharePath, Modified, ModifiedBy, SharePointSite and doclib and Docsubfolder.
This matches the columns required by the Migration Manger as documented at https://learn.microsoft.com/SharePointmigration/mm-bulk-upload-format-csv-json. (Note that Modified, ModifiedBy are not used by the tool, they are just used as filler). You can then export this list as a csv, remove all rows that dont have a valid SharePointSite and doclib and Docsubfolder and upload it to the Migration Manager.

A flow can also be created to automatically trigger a migration when the SharePointSite and doclib and Docsubfolder are updated. The flow needs use 'Send and Http Request to SharePoint' to Post Data to tenant-admin.SharePoint.com/_api/MigrationCenterServices/Tasks/BatchCreate.
The body of the request should contain 
```json
{
    "taskSettings": {
        "AgentGroupName": "",
        "AzureActiveDirectoryLkp": true,
        "CustomAzureAccessKey": "",
        "CustomAzureDeletionAfterMig": false,
        "CustomAzureStorageAccount": "",
        "DateCreated": "2020-01-26T17:42:25.634Z",
        "DateModified": "2020-01-26T17:42:25.634Z",
        "EnableIncremental": false,
        "EnableUserMappings": false,
        "Encrypted": true,
        "FilterOutHiddenFiles": false,
        "FilterOutPathSpecialCharacters": false,
        "IgnoredFileExtensions": "",
        "InvalidCharsReplacement": "",
        "MigrateAllWebStructures": false,
        "MigrateOneNoteNotebook": true,
        "MigrateSchema": true,
        "PreservePermissionForFileShare": false,
        "PreserveUserPermissionForOnPrem": true,
        "ReplaceInvalidChars": false,
        "ScanOnly": false,
        "SkipListWithAudienceEnabled": true,
        "StartMigrationAutomaticallyWhenNoScanIssue": false,
        "Tags": [],
        "TurnOnDateCreatedFilter": false,
        "TurnOnDateModifiedFilter": false,
        "TurnOnExtensionFilter": false,
        "UseCustomAzureStorage": false,
        "UserMappingCSVFile": "",
        "VersionNumsPreserved": 10
    },
    "taskDefinitions": [
        {
            "Name": "testmig",
            "Type": 0,
            "SourceUri": "\\\\server\\share\\folder",
            "SourceListName": "",
            "SourceListRelativePath": "",
            "TargetSiteUrl": "https://tenant.SharePoint.com/sites/site",
            "TargetListName": "testlib",
            "TargetListRelativePath": "folder"
        }
     
    ],
    "mmTaskSettings": {
        "ScheduledType": 0,
        "ScheduledTimeUtc": "1901-01-02T00:00:00.000Z",
        "AgentGroupName": "Default"
    }
}

```

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "Local directory path to scan (e.g., C:\\FileShares\\HR)")]
    [ValidateScript({ Test-Path $_ -PathType Container })]
    [string]$RootPath,
    
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL (e.g., https://contoso.sharepoint.com/sites/migration)")]
    [ValidatePattern('^https://')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory = $true, HelpMessage = "Target SharePoint list name")]
    [string]$ListName,
    
    [Parameter(HelpMessage = "Maximum directory depth to scan (default: 20)")]
    [ValidateRange(1, 20)]
    [int]$MaxDepth = 20,
    
    [Parameter(HelpMessage = "Batch size for list item creation (default: 500)")]
    [ValidateRange(1, 1000)]
    [int]$BatchSize = 500
)

begin {
    $script:Summary = @{
        FoldersScanned = 0
        ItemsCreated = 0
        PathsSkipped = 0
        Failures = 0
    }
    
    Write-Verbose "Ensuring user is logged in to CLI for Microsoft 365..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' first."
    }
    
    Write-Verbose "Validating target SharePoint list: $ListName"
    $listJson = m365 spo list get --webUrl $SiteUrl --title $ListName --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "List '$ListName' not found at $SiteUrl. Please create the list with required columns (Title, Level, Level1-Level20) before running this script."
    }
    
    Write-Verbose "List validated successfully. Beginning directory scan..."
}

process {
    try {
        Write-Verbose "Scanning directory structure: $RootPath"
        
        $allFolders = Get-ChildItem -Path $RootPath -Directory -Recurse -ErrorAction SilentlyContinue |
            Where-Object { $_.FullName.Length -le 260 }
        
        $skippedPaths = Get-ChildItem -Path $RootPath -Directory -Recurse -ErrorAction SilentlyContinue |
            Where-Object { $_.FullName.Length -gt 260 }
        
        $script:Summary.FoldersScanned = $allFolders.Count
        $script:Summary.PathsSkipped = $skippedPaths.Count
        
        if ($skippedPaths.Count -gt 0) {
            Write-Warning "Skipped $($skippedPaths.Count) folder(s) with paths exceeding 260 characters (SharePoint limit)"
        }
        
        Write-Verbose "Found $($allFolders.Count) folder(s) to process"
        
        if ($allFolders.Count -eq 0) {
            Write-Warning "No folders found in $RootPath"
            return
        }
        
        $items = @()
        foreach ($folder in $allFolders) {
            $relativePath = $folder.FullName.Replace($RootPath, "").TrimStart('\\')
            $parts = if ($relativePath) { $relativePath.Split('\\') } else { @() }
            $depth = $parts.Count
            
            if ($depth -gt $MaxDepth) {
                Write-Warning "Folder depth ($depth) exceeds MaxDepth ($MaxDepth): $($folder.FullName)"
                $script:Summary.PathsSkipped++
                continue
            }
            
            $row = [PSCustomObject]@{
                Title = $folder.FullName
                Level = $depth
            }
            
            for ($i = 1; $i -le 20; $i++) {
                $levelValue = if ($i -le $parts.Count) { $parts[$i - 1] } else { "" }
                $row | Add-Member -MemberType NoteProperty -Name "Level$i" -Value $levelValue
            }
            
            $items += $row
        }
        
        Write-Verbose "Prepared $($items.Count) list item(s) for batch creation"
        
        $totalBatches = [Math]::Ceiling($items.Count / $BatchSize)
        $currentBatch = 0
        
        for ($i = 0; $i -lt $items.Count; $i += $BatchSize) {
            $currentBatch++
            $batchItems = $items[$i..[Math]::Min($i + $BatchSize - 1, $items.Count - 1)]
            
            Write-Progress -Activity "Creating list items" -Status "Batch $currentBatch of $totalBatches" -PercentComplete (($currentBatch / $totalBatches) * 100)
            
            $csvContent = ($batchItems | ConvertTo-Csv -NoTypeInformation) -join "`n"
            $csvContent = $csvContent.Replace('"', '\"')
            
            if ($PSCmdlet.ShouldProcess("Batch $currentBatch ($($batchItems.Count) items)", "Create list items")) {
                try {
                    m365 spo listitem batch add --webUrl $SiteUrl --listTitle $ListName --csvContent $csvContent 2>&1 | Out-Null
                    
                    if ($LASTEXITCODE -eq 0) {
                        $script:Summary.ItemsCreated += $batchItems.Count
                        Write-Verbose "  Batch $currentBatch: Created $($batchItems.Count) item(s)"
                    } else {
                        Write-Warning "Batch $currentBatch failed to create items"
                        $script:Summary.Failures += $batchItems.Count
                    }
                } catch {
                    Write-Warning "Error processing batch $currentBatch: $($_.Exception.Message)"
                    $script:Summary.Failures += $batchItems.Count
                }
            } else {
                $script:Summary.ItemsCreated += $batchItems.Count
            }
        }
        
        Write-Progress -Activity "Creating list items" -Completed
        
    } catch {
        Write-Warning "Error during directory scan: $($_.Exception.Message)"
        $script:Summary.Failures++
    }
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "  Directory to SharePoint List Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Folders Scanned     : $($Summary.FoldersScanned)" -ForegroundColor White
    Write-Host "Items Created       : $($Summary.ItemsCreated)" -ForegroundColor Green
    
    if ($Summary.PathsSkipped -gt 0) {
        Write-Host "Paths Skipped       : $($Summary.PathsSkipped)" -ForegroundColor Yellow
    }
    
    if ($Summary.Failures -gt 0) {
        Write-Host "Failures            : $($Summary.Failures)" -ForegroundColor Red
    }
    
    Write-Host "========================================" -ForegroundColor Cyan
    
    if ($Summary.Failures -eq 0 -and $Summary.ItemsCreated -gt 0) {
        Write-Host "`n✅ Script completed successfully!" -ForegroundColor Green
        Write-Host "Next steps:" -ForegroundColor White
        Write-Host "  1. Open the list at: $SiteUrl/Lists/$($ListName.Replace(' ', ''))" -ForegroundColor Gray
        Write-Host "  2. Fill in SharePointSite, DocLibrary, and DocSubfolder columns" -ForegroundColor Gray
        Write-Host "  3. Export to CSV for SharePoint Migration Manager" -ForegroundColor Gray
    } elseif ($Summary.Failures -gt 0) {
        Write-Host "`n⚠️ Script completed with errors. Review warnings above." -ForegroundColor Yellow
    }
}

# Usage examples:
# Basic usage - scan directory and create list items
# .\Copy-DirectoryToList.ps1 -RootPath "C:\FileShares\HR" -SiteUrl "https://contoso.sharepoint.com/sites/migration" -ListName "MigrationFolders"

# Limit scan depth to 10 levels
# .\Copy-DirectoryToList.ps1 -RootPath "C:\FileShares\Finance" -SiteUrl "https://contoso.sharepoint.com/sites/migration" -ListName "MigrationFolders" -MaxDepth 10

# Preview what would be created (WhatIf mode)
# .\Copy-DirectoryToList.ps1 -RootPath "C:\FileShares\IT" -SiteUrl "https://contoso.sharepoint.com/sites/migration" -ListName "MigrationFolders" -WhatIf

# Verbose output with custom batch size
# .\Copy-DirectoryToList.ps1 -RootPath "C:\FileShares\Sales" -SiteUrl "https://contoso.sharepoint.com/sites/migration" -ListName "MigrationFolders" -BatchSize 250 -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
CLS
$count = 0
$rootpath = "\\server\fileshare"
Connect-PnPOnline -Url "https://tenant.SharePoint.com/sites/site with the targetlist" -Interactive
$list = Get-PnPList  "FolderTest4"
$Batch = new-PnPBatch
function add-listitemwithLevels {
    PARAM (
        [PARAMETER(Mandatory = $True, Position = 0, HelpMessage = "Path")][String]$path,
        [PARAMETER(Mandatory = $True, Position = 0, HelpMessage = "Level")][String]$level,
        [PARAMETER(Mandatory = $True, Position = 0, HelpMessage = "level1")][String]$level1,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level2")][String]$level2,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level3")][String]$level3,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level4")][String]$level4,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level5")][String]$level5,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level6")][String]$level6,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level7")][String]$level7,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level8")][String]$level8,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level9")][String]$level9,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level10")][String]$level10,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level11")][String]$level11,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level12")][String]$level12,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level13")][String]$level13,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level14")][String]$level14,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level15")][String]$level15,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level16")][String]$level16,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level17")][String]$level17,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level18")][String]$level18,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level19")][String]$level19,
        [PARAMETER(Mandatory = $false, Position = 0, HelpMessage = "level20")][String]$level20
    
    )
   
    Add-PnPListItem -List $list -Values     @{"Title" = $path; "Level" = $level; "Level1" = $level1; "Level2" = $level2; "Level3" = $level3; "Level4" = $level4; "Level5" = $level5; "Level6" = $level6; "Level7" = $level7; "Level8" = $level8; "Level9" = $level9; "Level10" = $level10; "Level11" = $level11; "Level12" = $level12; "Level13" = $level13; "Level14" = $level14; "Level15" = $level15; "Level16" = $level16; "Level17" = $level17; "Level18" = $level18; "Level19" = $level19; "Level20" = $level20 } -Batch $Batch
    $Global:count = $Global:count + 1
    if ( $Global:count -eq 500) {
        Invoke-PnpBatch $Batch
        $Batch = new-PnPBatch
        $Global:count = 0
    }

}
function get-folders {
    PARAM (
        [PARAMETER(Mandatory = $True, Position = 0, HelpMessage = "Path")][String]$path,
        [PARAMETER(Mandatory = $True, Position = 0, HelpMessage = "Level")][Int32]$level
    )

    $items = Get-ChildItem $path -Directory
    foreach ($item in $items) {
        if ($item.Mode -eq "d-----") {
            $itemname = $item.Name
            $fullpath = "$path\$itemname"
            switch ($level) {
                1 { $level1 = $itemname }
                2 { $level2 = $itemname }
                3 { $level3 = $itemname }
                4 { $level4 = $itemname }
                5 { $level5 = $itemname }
                6 { $level6 = $itemname }
                7 { $level7 = $itemname }
                8 { $level8 = $itemname }
                9 { $level9 = $itemname }
                10 { $level10 = $itemname }
                11 { $level11 = $itemname }
                12 { $level12 = $itemname }
                13 { $level13 = $itemname }
                14 { $level14 = $itemname }
                15 { $level15 = $itemname }
                16 { $level16 = $itemname }
                17 { $level17 = $itemname }
                18 { $level18 = $itemname }
                19 { $level19 = $itemname }
                20 { $level20 = $itemname }
                Default { Write-Host "Level $level reached" }
            }
            add-listitemwithLevels -path $fullpath -level $level  -level1 $level1  -level2 $level2  -level3 $level3  -level4 $level4  -level5 $level5  -level6 $level6  -level7 $level7  -level8 $level8  -level9 $level9  -level10 $level10  -level11 $level11  -level12 $level12  -level13 $level13  -level14 $level14 -level15 $level15  -level16 $level16  -level17 $level17  -level18 $level18  -level19 $level19  -level20 $level20 
            get-folders -path $fullpath -level ($level + 1) 
        }

    }
}

$level1 = ""
$level2 = ""
$level3 = ""
$level4 = ""
$level5 = ""
$level6 = ""
$level7 = ""
$level8 = ""
$level9 = ""
$level10 = ""
$level11 = ""
$level12 = ""
$level13 = ""
$level14 = ""
$level15 = ""
$level16 = ""
$level17 = ""
$level18 = ""
$level19 = ""
$level20 = ""

$subfolders = get-folders -path $rootpath -level 1
 if ( $Global:count-gt 0){
  Invoke-PnpBatch $Batch
}

    # End

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| Russell Gove |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-copy-directory-structure-to-sharepoint-list" aria-hidden="true" />

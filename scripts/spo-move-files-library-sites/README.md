

# Copying files between different SharePoint libraries with custom metadata

You might have a requirement to move sample files from a site to a different site, e.g. subset of production files to UAT site to allow testing of solutions. You may want better control over metadata settings, such as ProcessStatus, ensuring files are marked as "Pending" upon transfer. Unlike the default file copy feature, this script enables you to skip the copy process if the destination site lacks a matching folder structure as well as setting custom metadata to specific values.

This sample shows how to achieve this using both PnP PowerShell and CLI for Microsoft 365 (v11.4.0+).

## Summary

# [PnP PowerShell](#tab/pnpps)

```PowerShell

﻿param (
    [Parameter(Mandatory=$false)]
    [string]$SourceSiteUrl = "https://contoso.sharepoint.com/teams/app",
    [Parameter(Mandatory=$false)]
    [string]$SourceFolderPath=  "https://contoso.sharepoint.com/teams/app/Temp Library/test",
    [Parameter(Mandatory=$false)]
    [string]$DestinationSiteUrl = "https://contoso.sharepoint.com/teams/t-app",
    [Parameter(Mandatory=$false)]
    [string]$DestinationFolderPath = "https://contoso.sharepoint.com/teams/t-app/TempLibrary/test"
)

# Generate a unique log file name using today's date
$todayDate = Get-Date -Format "yyyy-MM-dd"
$logFileName = "CopyFilesToSharePoint_$todayDate.log"
$logFilePath = Join-Path -Path $PSScriptRoot -ChildPath $logFileName

# Connect to the source and destination SharePoint sites
Connect-PnPOnline -Url $SourceSiteUrl -Interactive
$SourceConn  = Get-PnPConnection 
Connect-PnPOnline -Url $DestinationSiteUrl -Interactive
$DestConn  = Get-PnPConnection 
# Function to copy files recursively and log errors
function Copy-FilesToSharePoint {
    param (
        [string]$SourceFolderPath,
        [string]$DestinationFolderPath
    )
    $sourceRelativeFolderPath = $SourceFolderPath.Replace($SourceSiteUrl,'') 
    $sourceFiles = Get-PnPFolderItem  -FolderSiteRelativeUrl $sourceRelativeFolderPath -ItemType File -Connection $SourceConn
    foreach ($file in $sourceFiles) {
        $relativePath = $file.ServerRelativePath
       
        # Check if the destination folder exists
        $destinationFolder = Get-PnPFolder -Url $DestinationFolderPath -Connection $DestConn -ErrorAction SilentlyContinue
        if ($null -eq $destinationFolder) {
            $errorMessage = "Error: Destination folder '$DestinationFolderPath' does not exist."
            Write-Host $errorMessage -ForegroundColor Red
            Add-Content -Path $logFilePath -Value $errorMessage
            continue
        }

        try {
            #get file as stream
           $fileUrl =  $SourceFolderPath + "/" + $file.Name
           $p = $fileUrl.Replace($SourceSiteUrl,'') 
           $streamResult = Get-PnPFile -Url  $p  -Connection $SourceConn -AsMemoryStream
            # Upload the file to the destination folder
           $uploadedFile = Add-PnPFile -Folder $DestinationFolderPath -FileName $file.Name -Stream  $streamResult  -Values @{"ProcessStatus" = "Pending"} -Connection $DestConn #-ErrorAction St
       
            Write-Host "File '$($file.Name)' copied and status set to 'Pending' in '$DestinationFolderPath'" -ForegroundColor Green
        } catch {
            $errorMessage = "Error copying file '$($file.Name)' to '$DestinationFolderPath': $($_.Exception.Message)"
            Write-Host $errorMessage -ForegroundColor Red
            Add-Content -Path $logFilePath -Value $errorMessage
        }
    }
}


# Call the function to copy files to SharePoint
$sourceRelativeFolderPath = $SourceFolderPath.Replace($SourceSiteUrl,'') 
$sourceLevel1Folders = Get-PnPFolderItem  -FolderSiteRelativeUrl $sourceRelativeFolderPath -ItemType Folder  -Connection $SourceConn
Copy-FilesToSharePoint -SourceFolderPath $SourceFolderPath -DestinationFolderPath $DestinationFolderPath
$sourceLevel1Folders | ForEach-Object {
$sourceLevel1Folder = $_ 
if($_.Name -ne "Forms"){
    $sourcePath = $SourceFolderPath + "/" + $sourceLevel1Folder.Name
    $destPath = $DestinationFolderPath + "/" + $sourceLevel1Folder.Name
    Copy-FilesToSharePoint -SourceFolderPath $sourcePath  -DestinationFolderPath $destPath
    }
  $sourceLevel1Path =  $sourceRelativeFolderPath + "/" + $_.Name
  $sourceLevel2Folders = Get-PnPFolderItem  -FolderSiteRelativeUrl $sourceLevel1Path  -ItemType Folder  -Connection $SourceConn
  $sourceLevel2Folders | ForEach-Object {
    $sourceLevel2Folder = $_
    $sourcePath = $SourceFolderPath + "/" + $sourceLevel1Folder.Name + "/" + $sourceLevel2Folder.Name
    $destPath = $DestinationFolderPath + "/" + $sourceLevel1Folder.Name + "/" + $sourceLevel2Folder.Name
    Copy-FilesToSharePoint -SourceFolderPath $sourcePath  -DestinationFolderPath $destPath 
 }
}
# Disconnect from SharePoint
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "Source site URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SourceSiteUrl,
    
    [Parameter(Mandatory = $true, HelpMessage = "Source folder server-relative URL (e.g., /sites/app/TempLibrary/test)")]
    [ValidateNotNullOrEmpty()]
    [string]$SourceFolderUrl,
    
    [Parameter(Mandatory = $true, HelpMessage = "Destination site URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$DestinationSiteUrl,
    
    [Parameter(Mandatory = $true, HelpMessage = "Destination folder server-relative URL")]
    [ValidateNotNullOrEmpty()]
    [string]$DestinationFolderUrl,
    
    [Parameter(Mandatory = $false, HelpMessage = "Output directory for log file")]
    [string]$OutputPath = (Get-Location).Path,
    
    [Parameter(Mandatory = $false, HelpMessage = "Custom metadata field name to set")]
    [string]$MetadataFieldName = "ProcessStatus",
    
    [Parameter(Mandatory = $false, HelpMessage = "Custom metadata field value to set")]
    [string]$MetadataFieldValue = "Pending",
    
    [Parameter(Mandatory = $false, HelpMessage = "Maximum folder recursion depth (0 = unlimited)")]
    [ValidateRange(0, 10)]
    [int]$MaxDepth = 2
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    
    $script:Summary = @{
        FilesCopied = 0
        FoldersCopied = 0
        Failures = 0
        SkippedMissingDest = 0
    }
    
    $script:TempFolder = Join-Path $env:TEMP "spo-move-files-$(Get-Date -Format 'yyyyMMddHHmmss')"
    New-Item -Path $script:TempFolder -ItemType Directory -Force | Out-Null
    Write-Verbose "Created temp folder: $($script:TempFolder)"
    
    $logPath = Join-Path $OutputPath "CopyFilesToSharePoint_$(Get-Date -Format 'yyyy-MM-dd').log"
    Start-Transcript -Path $logPath
    
    Write-Host "Starting cross-site file copy operation..." -ForegroundColor Cyan
    Write-Host "Source: $SourceSiteUrl" -ForegroundColor Yellow
    Write-Host "Source Folder: $SourceFolderUrl" -ForegroundColor Yellow
    Write-Host "Destination: $DestinationSiteUrl" -ForegroundColor Yellow
    Write-Host "Destination Folder: $DestinationFolderUrl" -ForegroundColor Yellow
}

process {
    function Copy-FolderRecursive {
        param(
            [string]$SourceUrl,
            [string]$DestUrl,
            [int]$CurrentDepth = 0
        )
        
        if ($MaxDepth -gt 0 -and $CurrentDepth -ge $MaxDepth) {
            Write-Verbose "Max depth ($MaxDepth) reached, skipping: $SourceUrl"
            return
        }
        
        Write-Verbose "Processing folder: $SourceUrl (depth: $CurrentDepth)"
        
        try {
            Write-Verbose "Checking if destination folder exists: $DestUrl"
            $destCheckJson = m365 spo folder get --webUrl $DestinationSiteUrl --url $DestUrl --output json 2>$null
            if ($LASTEXITCODE -ne 0) {
                $errorMsg = "Destination folder does not exist: $DestUrl"
                Write-Warning $errorMsg
                Add-Content -Path $logPath -Value $errorMsg
                $script:Summary.SkippedMissingDest++
                return
            }
            
            Write-Verbose "Getting items from source folder: $SourceUrl"
            $itemsJson = m365 spo folder list --webUrl $SourceSiteUrl --parentFolderUrl $SourceUrl --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to list items in source folder: $SourceUrl"
                $script:Summary.Failures++
                return
            }
            
            $items = @($itemsJson | ConvertFrom-Json)
            if ($items.Count -eq 0) {
                Write-Verbose "No items found in: $SourceUrl"
                return
            }
            
            $files = $items | Where-Object { $_.ItemCount -eq $null }
            $folders = $items | Where-Object { $_.ItemCount -ne $null -and $_.Name -ne 'Forms' }
            
            Write-Host "  Found $($files.Count) files and $($folders.Count) folders in: $SourceUrl" -ForegroundColor Cyan
            
            $fileIndex = 0
            foreach ($file in $files) {
                $fileIndex++
                Write-Progress -Activity "Copying files" -Status "$($file.Name) ($fileIndex/$($files.Count))" -PercentComplete (($fileIndex / $files.Count) * 100)
                
                $tempFilePath = Join-Path $script:TempFolder $file.Name
                
                if ($PSCmdlet.ShouldProcess($file.Name, "Copy file to $DestUrl")) {
                    try {
                        Write-Verbose "Downloading file: $($file.ServerRelativeUrl)"
                        m365 spo file get --webUrl $SourceSiteUrl --url $file.ServerRelativeUrl --asFile --path $tempFilePath | Out-Null
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to download file: $($file.Name)"
                        }
                        
                        Write-Verbose "Uploading file to: $DestUrl"
                        $uploadJson = m365 spo file add --webUrl $DestinationSiteUrl --folder $DestUrl --path $tempFilePath --output json
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to upload file: $($file.Name)"
                        }
                        
                        $uploadedFile = $uploadJson | ConvertFrom-Json
                        Write-Verbose "Uploaded file ID: $($uploadedFile.ListItemAllFields.Id)"
                        
                        $destLibraryTitle = ($DestUrl -split '/')[-1]
                        Write-Verbose "Setting metadata: $MetadataFieldName = $MetadataFieldValue"
                        m365 spo listitem set --webUrl $DestinationSiteUrl --listTitle $destLibraryTitle --id $uploadedFile.ListItemAllFields.Id --$MetadataFieldName $MetadataFieldValue | Out-Null
                        if ($LASTEXITCODE -eq 0) {
                            Write-Host "    ✓ Copied: $($file.Name) (metadata: $MetadataFieldName=$MetadataFieldValue)" -ForegroundColor Green
                            $script:Summary.FilesCopied++
                        } else {
                            Write-Warning "File copied but failed to set metadata: $($file.Name)"
                            $script:Summary.FilesCopied++
                        }
                        
                        Remove-Item -Path $tempFilePath -Force -ErrorAction SilentlyContinue
                    }
                    catch {
                        $errorMsg = "Error copying file '$($file.Name)': $($_.Exception.Message)"
                        Write-Warning $errorMsg
                        Add-Content -Path $logPath -Value $errorMsg
                        $script:Summary.Failures++
                        Remove-Item -Path $tempFilePath -Force -ErrorAction SilentlyContinue
                        continue
                    }
                }
            }
            
            Write-Progress -Activity "Copying files" -Completed
            
            foreach ($folder in $folders) {
                $sourceSubFolder = "$SourceUrl/$($folder.Name)"
                $destSubFolder = "$DestUrl/$($folder.Name)"
                
                Write-Host "  Processing subfolder: $($folder.Name)" -ForegroundColor Yellow
                $script:Summary.FoldersCopied++
                
                Copy-FolderRecursive -SourceUrl $sourceSubFolder -DestUrl $destSubFolder -CurrentDepth ($CurrentDepth + 1)
            }
        }
        catch {
            $errorMsg = "Error processing folder '$SourceUrl': $($_.Exception.Message)"
            Write-Warning $errorMsg
            Add-Content -Path $logPath -Value $errorMsg
            $script:Summary.Failures++
        }
    }
    
    Copy-FolderRecursive -SourceUrl $SourceFolderUrl -DestUrl $DestinationFolderUrl -CurrentDepth 0
}

end {
    if (Test-Path $script:TempFolder) {
        Remove-Item -Path $script:TempFolder -Recurse -Force -ErrorAction SilentlyContinue
        Write-Verbose "Cleaned up temp folder: $($script:TempFolder)"
    }
    
    Write-Host "\n=== File Copy Operation Summary ===" -ForegroundColor Cyan
    Write-Host "Files Copied: " -NoNewline
    Write-Host $script:Summary.FilesCopied -ForegroundColor Green
    Write-Host "Folders Processed: " -NoNewline
    Write-Host $script:Summary.FoldersCopied -ForegroundColor Green
    Write-Host "Skipped (Missing Dest): " -NoNewline
    Write-Host $script:Summary.SkippedMissingDest -ForegroundColor Yellow
    Write-Host "Failures: " -NoNewline
    if ($script:Summary.Failures -gt 0) {
        Write-Host $script:Summary.Failures -ForegroundColor Red
    } else {
        Write-Host $script:Summary.Failures -ForegroundColor Green
    }
    Write-Host "Log File: " -NoNewline
    Write-Host $logPath -ForegroundColor Cyan
    Write-Host "====================================\n" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Example 1: Copy files between sites with default metadata
# .\\Copy-FilesBetweenLibraries.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/app" -SourceFolderUrl "/sites/app/TempLibrary/test" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/t-app" -DestinationFolderUrl "/sites/t-app/TempLibrary/test"

# Example 2: Copy with WhatIf to preview operations
# .\\Copy-FilesBetweenLibraries.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/app" -SourceFolderUrl "/sites/app/TempLibrary/test" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/t-app" -DestinationFolderUrl "/sites/t-app/TempLibrary/test" -WhatIf

# Example 3: Copy with custom metadata field
# .\\Copy-FilesBetweenLibraries.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/app" -SourceFolderUrl "/sites/app/TempLibrary" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/uat" -DestinationFolderUrl "/sites/uat/Documents" -MetadataFieldName "Status" -MetadataFieldValue "Review"

# Example 4: Unlimited recursion with verbose output
# .\\Copy-FilesBetweenLibraries.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/app" -SourceFolderUrl "/sites/app/TempLibrary" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/t-app" -DestinationFolderUrl "/sites/t-app/Documents" -MaxDepth 0 -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Reshmee Auckloo |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-move-files-library-sites" aria-hidden="true" />

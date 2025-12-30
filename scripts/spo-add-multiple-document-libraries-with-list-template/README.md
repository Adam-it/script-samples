

# Create multiple document libraries using custom list template

## Summary

  [Creating custom list templates](https://learn.microsoft.com/sharepoint/lists-custom-template) is now possible to create both custom document libraries and lists although official microsoft documentation has not specified anything about supporting custom document library templates. This script will create multiple instances of document library by applying custom list template. Please refer to [Create and add list template to SharePoint site with content types,site columns and list views](https://pnp.github.io/script-samples/spo-add-list-template-with-custom-library/README.html) to add a custom list template. However there are some limitations with the list design to  set permissions, apply versionings, create indexed columns , etc.. The PnP PowerShell version calls the cmdlet Invoke-SPOListDesign iteratively and amends the document library url and display name before applying versioning settings and creating indexed columns. The CLI for Microsoft 365 version creates libraries individually with versioning and navigation support.
 
More about list template 
 [https://learn.microsoft.com/sharepoint/lists-custom-template](https://learn.microsoft.com/sharepoint/lists-custom-template)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL")]
    [ValidatePattern('^https://')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory = $true, HelpMessage = "Path to CSV file with library definitions")]
    [string]$CsvPath,
    
    [Parameter(HelpMessage = "GUID of custom list template feature to apply")]
    [string]$TemplateFeatureId,
    
    [Parameter(HelpMessage = "Enable major and minor versioning")]
    [switch]$EnableVersioning,
    
    [Parameter(HelpMessage = "Maximum number of major versions to retain")]
    [int]$MajorVersionLimit = 500,
    
    [Parameter(HelpMessage = "Maximum number of minor versions to retain")]
    [int]$MinorVersionLimit = 10
)

begin {
    $currentTime = $(Get-Date).ToString("yyyyMMddHHmmss")
    $logFilePath = ".\\log-$currentTime.log"
    Start-Transcript -Path $logFilePath
    
    $script:Summary = @{
        LibrariesProcessed = 0
        LibrariesCreated   = 0
        Failures           = 0
    }

    Write-Verbose "Validating CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' manually."
    }

    Write-Verbose "Validating CSV file path..."
    if (-not (Test-Path $CsvPath)) {
        throw "CSV file not found: $CsvPath"
    }

    Write-Verbose "Importing CSV file..."
    $libraries = Import-Csv -Path $CsvPath
    
    if ($libraries.Count -eq 0) {
        throw "CSV file is empty or has no valid rows."
    }
    
    if (-not ($libraries[0].PSObject.Properties.Name -contains 'DisplayName')) {
        throw "CSV must contain 'DisplayName' column."
    }
    
    Write-Verbose "Found $($libraries.Count) librar$(if($libraries.Count -eq 1){'y'}else{'ies'}) to create."
}

process {
    $libCount = 0
    
    foreach ($library in $libraries) {
        $libCount++
        $script:Summary.LibrariesProcessed++
        
        $displayName = $library.DisplayName.Trim()
        
        Write-Progress -Activity "Creating Document Libraries" -Status "Processing: $displayName" -PercentComplete (($libCount / $libraries.Count) * 100)
        
        Write-Verbose "Processing library $libCount of $($libraries.Count): $displayName"
        
        try {
           if ($PSCmdlet.ShouldProcess($displayName, 'Create document library')) {
               Write-Verbose "  Creating library: $displayName"
               
                $createArgs = @('spo', 'list', 'add', '--webUrl', $SiteUrl, '--title', $displayName, '--baseTemplate', 'DocumentLibrary', '--output', 'json')
                if ($TemplateFeatureId) {
                    $createArgs += @('--templateFeatureId', $TemplateFeatureId)
                }
                $createResult = m365 @createArgs 2>&1
                
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to create library '$displayName'. CLI: $createResult"
                    $script:Summary.Failures++
                    continue
                }
                
                $createdList = $createResult | ConvertFrom-Json
                $listId = $createdList.Id
                
                Write-Verbose "  Library created with ID: $listId"
                
                Write-Verbose "  Waiting for library to be fully provisioned..."
                $maxRetries = 6
                $retryCount = 0
                $libraryVerified = $false
                
                while ($retryCount -lt $maxRetries -and -not $libraryVerified) {
                    Start-Sleep -Seconds 5
                    $verifyResult = m365 spo list get --webUrl $SiteUrl --id $listId --output json 2>&1
                    if ($LASTEXITCODE -eq 0) {
                        $libraryVerified = $true
                        Write-Verbose "  Library verified as provisioned"
                    } else {
                        $retryCount++
                        Write-Verbose "  Retry $retryCount/$maxRetries: Library not yet available"
                    }
                }
                
                if (-not $libraryVerified) {
                    Write-Warning "Library '$displayName' created but could not be verified. Skipping additional configuration."
                    $script:Summary.Failures++
                    continue
                }
                
                if ($EnableVersioning) {
                    Write-Verbose "  Configuring versioning settings..."
                    $versionResult = m365 spo list set --webUrl $SiteUrl --id $listId --enableVersioning true --enableMinorVersions true --majorVersionLimit $MajorVersionLimit --majorWithMinorVersionsLimit $MinorVersionLimit 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to enable versioning for '$displayName'. CLI: $versionResult"
                    } else {
                        Write-Verbose "  Versioning enabled (Major: $MajorVersionLimit, Minor: $MinorVersionLimit)"
                    }
                }
                
                Write-Verbose "  Adding to Quick Launch navigation..."
                $navUrl = "$($createdList.RootFolder.ServerRelativeUrl)/"
                $navResult = m365 spo navigation node add --webUrl $SiteUrl --location QuickLaunch --title $displayName --url $navUrl 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to add navigation node for '$displayName'. CLI: $navResult"
                }
                
                $script:Summary.LibrariesCreated++
                Write-Verbose "Successfully created library: $displayName"
            }
        } catch {
            Write-Warning "Unexpected error creating library '$displayName': $_"
            $script:Summary.Failures++
            continue
        }
    }
    
    Write-Progress -Activity "Creating Document Libraries" -Completed
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Library Creation Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "  Libraries Processed : $($Summary.LibrariesProcessed)" -ForegroundColor White
    Write-Host "  Libraries Created   : $($Summary.LibrariesCreated)" -ForegroundColor Green
    
    if ($Summary.Failures -gt 0) {
        Write-Host "  Failures            : $($Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "  Failures            : $($Summary.Failures)" -ForegroundColor Green
    }
    
    Write-Host "========================================" -ForegroundColor Cyan
    
    if ($EnableVersioning) {
        Write-Host "\nVersioning Settings:" -ForegroundColor Yellow
        Write-Host "  Major Versions: $MajorVersionLimit" -ForegroundColor White
        Write-Host "  Minor Versions: $MinorVersionLimit" -ForegroundColor White
    }
    
    if ($Summary.Failures -gt 0) {
        Write-Host "\nNote: Some libraries failed to create. Check warnings above for details." -ForegroundColor Yellow
    }
    
    Stop-Transcript
}

# Usage Examples:
# .\\Create-Libraries.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -CsvPath ".\\libraries.csv"
# .\\Create-Libraries.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -CsvPath ".\\libraries.csv" -EnableVersioning -MajorVersionLimit 100 -MinorVersionLimit 5
# .\\Create-Libraries.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -CsvPath ".\\libraries.csv" -TemplateFeatureId "5b38e500-0fab-4da7-b011-ad7113228920" -WhatIf
# .\\Create-Libraries.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -CsvPath ".\\libraries.csv" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
[CmdletBinding()] 
    Param(
    [Parameter(Mandatory=$false,  Position=0)]
    [String]$adminSiteUrl = "https://<tenant>-admin.sharepoint.com",
    [Parameter(Mandatory=$false,  Position=1)]
    [String]$siteUrl =  "https://<tenant>.sharepoint.com/sites/investment",
    [Parameter(Mandatory=$false,  Position=2)]
    [String]$librariesCSV =  "C:\\Scripts\\DocumentLibraryTemplate\\libraries.csv",

    [Parameter(Mandatory=$false,  Position=4)]
    [String]$listDesignId = "5b38e500-0fab-4da7-b011-ad7113228920" # use Get-SPOListDesign to find the Id of the list design containing the document library template
  )
#creating indexed columns might help with performance of large libraries, i.e. >5000 files
function Create-Index ($list, $targetFieldName)
{
  $targetField = Get-PnPField -List $list -Identity $targetFieldName
  $targetField.Indexed = 1;
  $targetField.Update();
  $list.Context.ExecuteQuery();
}

# log file will be saved in same directory script was started from  
$currentTime= $(get-date).ToString("yyyyMMddHHmmss")  
$logFilePath=".\\log-"+$currentTime+".log"  


## Start the Transcript  
Start-Transcript -Path $logFilePath 

Connect-SPOService $adminSiteUrl 
Connect-PnPOnline -Url $siteUrl -Interactive
Import-Csv $librariesCSV | ForEach-Object {
Invoke-SPOListDesign -Identity $listDesignId -WebUrl $siteUrl
#Get library just created and update Internal name and display name, replace <listName> with the name specified in the custom list template
$lib = Get-PnPList -Identity "<listName>" -Includes RootFolder
#wait until document library has been created
while(!$lib)
{
 $lib = Get-PnPList -Identity "<listName>" -Includes RootFolder
 sleep -second 5
}
if($lib)
{
    $lib.Rootfolder.MoveTo($($_.InternalName))  
    Invoke-PnPQuery  
    #this will change library title  
    Set-PnPList -Identity $lib.Id -Title $($_.DisplayName)
    #add document library to quick launch
    Add-PnPNavigationNode -Title $_.DisplayName -Url $($_.InternalName + "/") -Location "QuickLaunch"
    #enable versioning on the library
    Set-PnPList -Identity $lib.Id -EnableVersioning $True -EnableMinorVersions $True -MajorVersions 500 -MinorVersions 10
    Write-host "`tSetting versioning to major/minor to :"$_.DisplayName
    Create-Index $lib "Created By"
    Create-Index $lib "Modified"
 }
}


## Disconnect the context  
Disconnect-PnPOnline  
 
## Stop Transcript  
Stop-Transcript  

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CSV file sample](#tab/csv)

```csv
InternalName,DisplayName
AR,Annual Reports
CR,Credit Risk
```
***

## Results running the script 

![Example Screenshot](assets/example.png)


## Source Credit

Inspired by [Invoke-SPOListDesign to create instances of lists/libraires](https://reshmeeauckloo.wordpress.com/2021/10/27/invoke-spolistdesign-to-create-instances-of-lists-libraires/)

## Contributors

| Author(s) |
|-----------|
| Reshmee Auckloo |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-add-multiple-document-libraries-with-list-template" aria-hidden="true" />

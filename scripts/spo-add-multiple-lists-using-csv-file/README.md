

# Create list and libraries from CSV file

## Summary

This script sample will bulk create SharePoint lists or libraries using CLI for Microsoft 365 (v11.4.0+) from a CSV file with columns: Title, Template, Url.

## Implementation

- Open Windows PowerShell ISE
- Edit Script and add required parameters for Site URL and path to CSV file
- Press run

# [PnP PowerShell](#tab/pnpps)
```powershell

###### Declare and Initialize Variables ######  

#Destination site collection url
$url="https://<tenant>.sharepoint.com/sites/yoursite"

#Path to CSV file
$csvFilePath = "ListsAndLibraries.csv"


# log file will be saved in same directory script was started from
$saveDir = (Resolve-path ".\")  
$currentTime= $(get-date).ToString("yyyyMMddHHmmss")  
$logFilePath=".\log-"+$currentTime+".log"  

## Start the Transcript  
Start-Transcript -Path $logFilePath 



## Connect to SharePoint Online site  
Connect-PnPOnline -Url $Url -Interactive

## Import CSV file
$data = Import-Csv -Path $csvFilePath -Delimiter ";"

## Create list or library
$data | Foreach-Object{
   
   New-PnPList -Title $_.Title -Url $_.Url -Template $_.Template -OnQuickLaunch -EnableContentTypes 
   
}  
 
## Disconnect the context  
Disconnect-PnPOnline  
 
## Stop Transcript  
Stop-Transcript  
  

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory, HelpMessage = "Path to CSV file with list definitions (columns: Title;Template;Url)")]
    [ValidateScript({ Test-Path $_ -PathType Leaf })]
    [string]$CsvFilePath,
    
    [Parameter(HelpMessage = "Path where transcript log will be saved")]
    [ValidateNotNullOrEmpty()]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "create-lists-$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Yellow
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to authenticate to Microsoft 365"
    }
    
    Write-Host "Validating CSV file..." -ForegroundColor Yellow
    if (-not (Test-Path $CsvFilePath -PathType Leaf)) {
        Stop-Transcript
        throw "CSV file not found: $CsvFilePath"
    }
    
    try {
        $csvData = Import-Csv -Path $CsvFilePath -Delimiter ";"
    }
    catch {
        Stop-Transcript
        throw "Failed to import CSV file. Error: $_"
    }
    
    $requiredColumns = @('Title', 'Template', 'Url')
    $csvColumns = $csvData[0].PSObject.Properties.Name
    $missingColumns = $requiredColumns | Where-Object { $_ -notin $csvColumns }
    if ($missingColumns.Count -gt 0) {
        Stop-Transcript
        throw "CSV file is missing required columns: $($missingColumns -join ', '). Expected columns: Title;Template;Url"
    }
    
    $script:Summary = @{
        TotalLists    = $csvData.Count
        ListsCreated  = 0
        Failures      = 0
    }
    
    Write-Host "Found $($csvData.Count) lists to create." -ForegroundColor Cyan
}

process {
    $counter = 0
    foreach ($row in $csvData) {
        $counter++
        $listTitle = $row.Title
        $listTemplate = $row.Template
        $listUrl = $row.Url
        
        $percentComplete = [int](($counter / $csvData.Count) * 100)
        Write-Progress -Activity "Creating SharePoint lists" -Status "Processing $listTitle ($counter of $($csvData.Count))" -PercentComplete $percentComplete
        
        if ($PSCmdlet.ShouldProcess($listTitle, 'Create SharePoint list')) {
            try {
                Write-Host "Creating list: $listTitle (Template: $listTemplate)" -ForegroundColor White
                m365 spo list add --title $listTitle --baseTemplate $listTemplate --webUrl $SiteUrl --output json | Out-Null
                
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to create list '$listTitle'. CLI returned exit code $LASTEXITCODE"
                    $script:Summary.Failures++
                    continue
                }
                
                Write-Host "  ✓ Successfully created list: $listTitle" -ForegroundColor Green
                $script:Summary.ListsCreated++
            }
            catch {
                Write-Warning "Error creating list '$listTitle': $_"
                $script:Summary.Failures++
                continue
            }
        }
    }
    
    Write-Progress -Activity "Creating SharePoint lists" -Completed
}

end {
    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Site URL: $SiteUrl" -ForegroundColor White
    Write-Host "Total Lists: $($Summary.TotalLists)" -ForegroundColor White
    Write-Host "Lists Created: $($Summary.ListsCreated)" -ForegroundColor Green
    Write-Host "Failures: $($Summary.Failures)" -ForegroundColor $(if ($Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Example 1: Basic usage
# .\Create-Lists.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/yoursite" -CsvFilePath "ListsAndLibraries.csv"

# Example 2: WhatIf mode to preview changes
# .\Create-Lists.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/yoursite" -CsvFilePath "ListsAndLibraries.csv" -WhatIf

# Example 3: Verbose mode with custom output path
# .\Create-Lists.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/yoursite" -CsvFilePath "ListsAndLibraries.csv" -OutputPath "C:\Logs" -Verbose

# Example 4: Confirm each operation
# .\Create-Lists.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/yoursite" -CsvFilePath "ListsAndLibraries.csv" -Confirm
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [CSV file](#tab/csv)
```csv
Title;Template;Url
PnP Library;DocumentLibrary;PnPLibrary
Announcements;Announcements;lists/Announcements
Custom Simple List;GenericList;lists/CustomSimpleList

```
***

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Valeras Narbutas |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-sharepoint-list-items-to-csv" aria-hidden="true" />

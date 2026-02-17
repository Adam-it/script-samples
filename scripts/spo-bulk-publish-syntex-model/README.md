

# Bulk Publish Syntex Models To Libraries

## Summary

This script sample will publish Syntex Document Understanding Models to many libraries. Available in both PnP PowerShell and CLI for Microsoft 365 v11.4.0+. *Currently only document understanding models can be templated and rolled out to many sites

## Implementation

- Create CSV file using the format supplied below and modify to reflect the values of your sites/libraries that you wish to deploy Syntex models to.
- Open Windows PowerShell ISE
- Edit Script and add required parameters for Syntex Content Center URL and path to CSV file
- Press run

# [PnP PowerShell](#tab/pnpps)
```powershell

###### Declare and Initialize Variables ######  

#Change To Reflect Your Syntex Content Center
$syntexContentCenter = "https://contoso.sharepoint.com/sites/HRContentCenter" 

#Path to CSV file
$csvFilePath = "Libraries.csv"

###### DO NOT EDIT BELOW THIS LINE #####

## log file will be saved in same directory script was started from
$saveDir = (Resolve-Path ".\")  
$currentTime= $(Get-Date).ToString("yyyyddMMHHmmss")  
$logFilePath=".\log-"+$currentTime+".log"  

## Start the Transcript  
Start-Transcript -Path $logFilePath 

## Connect to your Syntex Content Center
Connect-PnPOnline -Url $syntexContentCenter -Interactive

## Import CSV file
$libraries = Import-Csv -Path $csvFilePath -Delimiter ";"

## Create a new batch
$batch = New-PnPBatch

foreach($lib in $libraries) 
{ 

    $splatCmds = @{
        Model = $lib.Model
        TargetSiteUrl = $lib.TargetSiteUrl
        TargetWebServerRelativeUrl = $lib.TargetWebServerRelativeUrl
        TargetLibraryServerRelativeUrl = $lib.TargetLibraryServerRelativeUrl
        Batch = $batch
    }

    Publish-PnPSyntexModel @splatCmds

}

## Execute Batch - Add Syntex Model To Libraries
Invoke-PnPBatch -Batch $batch
 
## Disconnect the context  
Disconnect-PnPOnline  
 
## Stop Transcript  
Stop-Transcript  

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage="Syntex Content Center URL where models are stored")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)')]
    [string]$ContentCenterUrl,
    
    [Parameter(Mandatory, HelpMessage="Path to CSV file with library configurations")]
    [ValidateScript({Test-Path $_ -PathType Leaf})]
    [string]$CsvFilePath,
    
    [Parameter(HelpMessage="Output path for transcript and CSV report")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath -PathType Container)) {
            throw "Invalid OutputPath: $OutputPath. Path must exist and be a directory."
        }
    }
    
    $script:ReportCollection = [System.Collections.ArrayList]::new()
    $script:Summary = @{
        LibrariesProcessed = 0
        ModelsApplied = 0
        Failures = 0
    }
    
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = Join-Path $OutputPath "BulkPublishSyntexModel_$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Bulk Publishing Syntex Models to Libraries" -ForegroundColor Cyan
    Write-Host "Content Center: $ContentCenterUrl" -ForegroundColor Cyan
    Write-Host "CSV File: $CsvFilePath`n" -ForegroundColor Cyan
    
    try {
        $csvData = Import-Csv -Path $CsvFilePath -Delimiter ";"
        Write-Host "Found $($csvData.Count) libraries in CSV file`n" -ForegroundColor Green
    }
    catch {
        throw "Failed to import CSV file: $($_.Exception.Message)"
    }
    
    $requiredColumns = @("Model", "TargetSiteUrl", "TargetLibraryServerRelativeUrl")
    $csvColumns = $csvData[0].PSObject.Properties.Name
    $missingColumns = $requiredColumns | Where-Object { $_ -notin $csvColumns }
    
    if ($missingColumns.Count -gt 0) {
        throw "CSV missing required columns: $($missingColumns -join ', ')"
    }
}

process {
    foreach ($library in $csvData) {
        $script:Summary.LibrariesProcessed++
        
        try {
            Write-Verbose "Processing: Model='$($library.Model)' | Site='$($library.TargetSiteUrl)' | Library='$($library.TargetLibraryServerRelativeUrl)'"
            
            $targetDescription = "$($library.Model) to $($library.TargetSiteUrl)$($library.TargetLibraryServerRelativeUrl)"
            
            if ($PSCmdlet.ShouldProcess($targetDescription, "Apply Syntex model")) {
                m365 spp model apply --webUrl $library.TargetSiteUrl --contentCenterUrl $ContentCenterUrl --title $library.Model --listUrl $library.TargetLibraryServerRelativeUrl
                
                if ($LASTEXITCODE -eq 0) {
                    Write-Host "  ✓ Applied model '$($library.Model)' to $($library.TargetSiteUrl)$($library.TargetLibraryServerRelativeUrl)" -ForegroundColor Green
                    $script:Summary.ModelsApplied++
                    
                    $script:ReportCollection.Add([PSCustomObject]@{
                        Model = $library.Model
                        TargetSiteUrl = $library.TargetSiteUrl
                        TargetLibrary = $library.TargetLibraryServerRelativeUrl
                        Status = "Success"
                        ErrorMessage = ""
                    }) | Out-Null
                }
                else {
                    Write-Warning "  ✗ Failed to apply model '$($library.Model)' to $($library.TargetSiteUrl)$($library.TargetLibraryServerRelativeUrl)"
                    $script:Summary.Failures++
                    
                    $script:ReportCollection.Add([PSCustomObject]@{
                        Model = $library.Model
                        TargetSiteUrl = $library.TargetSiteUrl
                        TargetLibrary = $library.TargetLibraryServerRelativeUrl
                        Status = "Failed"
                        ErrorMessage = "CLI command returned exit code $LASTEXITCODE"
                    }) | Out-Null
                }
            }
        }
        catch {
            Write-Warning "  ✗ Error processing library: $($_.Exception.Message)"
            $script:Summary.Failures++
            
            $script:ReportCollection.Add([PSCustomObject]@{
                Model = $library.Model
                TargetSiteUrl = $library.TargetSiteUrl
                TargetLibrary = $library.TargetLibraryServerRelativeUrl
                Status = "Error"
                ErrorMessage = $_.Exception.Message
            }) | Out-Null
            
            continue
        }
    }
}

end {
    if ($script:ReportCollection.Count -gt 0) {
        $csvReportPath = Join-Path $OutputPath "SyntexModelReport_$timestamp.csv"
        $script:ReportCollection | Export-Csv -Path $csvReportPath -NoTypeInformation
        Write-Host "`nReport exported to: $csvReportPath" -ForegroundColor Green
    }
    
    Write-Host "`n=== Bulk Publish Summary ===" -ForegroundColor Cyan
    Write-Host "Libraries processed: $($script:Summary.LibrariesProcessed)" -ForegroundColor White
    Write-Host "Models applied: $($script:Summary.ModelsApplied)" -ForegroundColor White
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
    else {
        Write-Host "Failures: 0" -ForegroundColor Green
    }
    
    Stop-Transcript
}

# .\Publish-SyntexModelBulk.ps1 -ContentCenterUrl "https://contoso.sharepoint.com/sites/ContentCenter" -CsvFilePath ".\Libraries.csv"

# .\Publish-SyntexModelBulk.ps1 -ContentCenterUrl "https://contoso.sharepoint.com/sites/ContentCenter" -CsvFilePath ".\Libraries.csv" -WhatIf

# .\Publish-SyntexModelBulk.ps1 -ContentCenterUrl "https://contoso.sharepoint.com/sites/ContentCenter" -CsvFilePath ".\Libraries.csv" -Verbose

# .\Publish-SyntexModelBulk.ps1 -ContentCenterUrl "https://contoso.sharepoint.com/sites/ContentCenter" -CsvFilePath ".\Libraries.csv" -OutputPath "C:\\Reports"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [CSV file](#tab/csv)
```csv
Model;TargetSiteUrl;TargetWebServerRelativeUrl;TargetLibraryServerRelativeUrl
Aviation Incident Report;https://contoso.sharepoint.com/sites/Retail;/sites/Retail;/sites/Retail/shared%20documents
Refinement Rules Example;https://contoso.sharepoint.com/sites/SalesAndMarketing;/sites/SalesAndMarketing;/sites/SalesAndMarketing/shared%20documents


```
***

## Contributors

| Author(s) |
|-----------|
| [Leon Armston](https://github.com/LeonArmston) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-bulk-publish-syntex-model" aria-hidden="true" />

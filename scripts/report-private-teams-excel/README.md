

# Report of Private Teams channels to Excel

## Summary

This sample gathers all Teams private channels in your tenant and produces an Excel report which is uploaded to a SharePoint site. This sample has been modernized to use CLI for Microsoft 365 v11.4.0+ and follows modern PowerShell best practices including typed parameters, WhatIf support, error handling, and progress reporting.

The script:
- Retrieves all Teams private channel sites (webTemplate TEAMCHANNEL#0)
- Generates an Excel report with channel details (Title, URL, Storage, Owner, Sharing)
- Uploads the report to a specified SharePoint library
- Supports WhatIf mode for safe testing

![Example Screenshot](assets/example.png)

> [!Note]
> For this sample, you will require the [Excel PowerShell module](https://www.powershellgallery.com/packages/ImportExcel) to be installed

# [PnP PowerShell](#tab/pnpps)

```powershell

Write-Host "Running Script..."

# Connect to the standard SharePoint Site
$siteConn = Connect-PnPOnline -Url "https://contoso.sharepoint.com" -Interactive -ReturnConnection
    
# Connect to the SharePoint Online Admin Service
$adminSiteConn = Connect-PnPOnline -Url "https://contoso-admin.sharepoint.com" -Interactive -ReturnConnection

# SharePointy Adminy Stuff here
Write-Host "Connected to SharePoint Online Admin Center"
    
#-----------------
# Gather Reporting Data
#-----------------

# Gets all Team Private Channels based on the template
$teamPrivateChannels = Get-PnPTenantSite -Template "TEAMCHANNEL#0" -Connection $adminSiteConn
    
#-----------------
# Produce and Save Reporting Data
#-----------------
$now = [System.DateTime]::Now.ToString("yyyy-mm-dd_hh-MM-ss")
$reportFileName = "teams-private-channels-$($now).xlsx"

$ExcelReportSettings = @{
    Path          = $reportFileName
    Title         = "Teams Private Channel Report"
    WorksheetName = "Teams Private Channels"
    AutoFilter    = $true 
    AutoSize      = $true
}

Write-Host "Creating Excel File $reportFileName"
$teamPrivateChannels | Select-Object Title, Url, StorageUsage, Owner, SiteDefinedSharingCapability `
| Export-Excel @ExcelReportSettings

# Save to SharePoint
$file = Add-PnPFile -Path $reportFileName -Folder "Shared Documents" -Connection $siteConn

Write-Host "Uploaded Excel File to SharePoint"

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL where the Excel report will be uploaded")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Document library folder name for report upload")]
    [ValidateNotNullOrEmpty()]
    [string]$FolderName = "Shared Documents",

    [Parameter(Mandatory = $false, HelpMessage = "Local directory path where Excel report will be created")]
    [ValidateScript({ Test-Path $_ -PathType Container })]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = Join-Path $OutputPath "Report-PrivateTeamsChannels_$timestamp.log"
    Start-Transcript -Path $transcriptPath -Append

    Write-Host "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] Starting Teams Private Channels Report Generation" -ForegroundColor Cyan

    # Validate Excel module
    if (-not (Get-Module -ListAvailable -Name ImportExcel)) {
        throw "ImportExcel module is required. Install with: Install-Module -Name ImportExcel -Scope CurrentUser"
    }

    # Initialize summary tracking
    $script:Summary = @{
        ChannelsFound   = 0
        FileCreated     = $false
        FileUploaded    = $false
    }

    try {
        # Ensure CLI login
        Write-Verbose "Ensuring CLI for Microsoft 365 login..."
        m365 login --ensure
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to authenticate with CLI for Microsoft 365. Exit code: $LASTEXITCODE"
        }
        Write-Verbose "Successfully authenticated with CLI for Microsoft 365"

        # Retrieve Teams private channel sites
        Write-Host "Retrieving Teams private channel sites..." -ForegroundColor Yellow
        $sitesJson = m365 spo site list --webTemplate "TEAMCHANNEL#0" --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve Teams private channel sites. Exit code: $LASTEXITCODE"
        }

        $teamPrivateChannels = @($sitesJson | ConvertFrom-Json)
        $script:Summary.ChannelsFound = $teamPrivateChannels.Count
        Write-Host "Found $($script:Summary.ChannelsFound) Teams private channel(s)" -ForegroundColor Green

        if ($script:Summary.ChannelsFound -eq 0) {
            Write-Warning "No Teams private channels found. Exiting."
            return
        }

        # Create Excel report
        Write-Host "Generating Excel report..." -ForegroundColor Yellow
        $reportFileName = "teams-private-channels-$timestamp.xlsx"
        $reportFilePath = Join-Path $OutputPath $reportFileName

        $ExcelReportSettings = @{
            Path          = $reportFilePath
            Title         = "Teams Private Channel Report"
            WorksheetName = "Teams Private Channels"
            AutoFilter    = $true
            AutoSize      = $true
        }

        $teamPrivateChannels | Select-Object Title, Url, StorageUsage, Owner, SiteDefinedSharingCapability |
            Export-Excel @ExcelReportSettings

        if (Test-Path $reportFilePath) {
            $script:Summary.FileCreated = $true
            Write-Host "Excel report created: $reportFilePath" -ForegroundColor Green
        } else {
            throw "Failed to create Excel report at $reportFilePath"
        }

    } catch {
        Write-Error "Error in begin block: $_"
        throw
    }
}

process {
    # No per-item processing needed - single upload operation handled in end block
}

end {
    try {
        if ($script:Summary.FileCreated) {
            $reportFilePath = Join-Path $OutputPath "teams-private-channels-$timestamp.xlsx"

            if ($PSCmdlet.ShouldProcess($reportFilePath, "Upload to $SiteUrl/$FolderName")) {
                Write-Host "Uploading Excel report to SharePoint..." -ForegroundColor Yellow

                $uploadResult = m365 spo file add --webUrl $SiteUrl --folder $FolderName --path $reportFilePath --output json
                if ($LASTEXITCODE -eq 0) {
                    $script:Summary.FileUploaded = $true
                    $uploadedFile = $uploadResult | ConvertFrom-Json
                    Write-Host "Report uploaded successfully: $($uploadedFile.ServerRelativeUrl)" -ForegroundColor Green
                } else {
                    Write-Warning "Failed to upload report to SharePoint. Exit code: $LASTEXITCODE"
                }
            } else {
                Write-Host "[WhatIf] Would upload $reportFilePath to $SiteUrl/$FolderName" -ForegroundColor Cyan
            }
        }

    } catch {
        Write-Error "Error uploading file: $_"
        $script:Summary.FileUploaded = $false
    } finally {
        # Display summary
        Write-Host "`n========================================" -ForegroundColor Cyan
        Write-Host "  Teams Private Channels Report Summary" -ForegroundColor Cyan
        Write-Host "========================================" -ForegroundColor Cyan
        Write-Host "Channels Found    : " -NoNewline
        Write-Host $script:Summary.ChannelsFound -ForegroundColor $(if ($script:Summary.ChannelsFound -gt 0) { "Green" } else { "Yellow" })
        Write-Host "Excel Created     : " -NoNewline
        Write-Host $(if ($script:Summary.FileCreated) { "Yes" } else { "No" }) -ForegroundColor $(if ($script:Summary.FileCreated) { "Green" } else { "Red" })
        Write-Host "Uploaded to SP    : " -NoNewline
        Write-Host $(if ($script:Summary.FileUploaded) { "Yes" } else { "No (check WhatIf mode or errors)" }) -ForegroundColor $(if ($script:Summary.FileUploaded) { "Green" } else { "Yellow" })
        Write-Host "========================================`n" -ForegroundColor Cyan

        Write-Host "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] Script completed" -ForegroundColor Cyan
        Stop-Transcript
    }
}

# Usage examples:

# Basic usage - upload report to specific site
# .\ Report-PrivateTeamsChannels.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/TeamReports"

# Specify custom folder and output directory
# .\ Report-PrivateTeamsChannels.ps1 -SiteUrl "https://contoso.sharepoint.com" -FolderName "Reports/Teams" -OutputPath "C:\Reports"

# Test with WhatIf (no file upload)
# .\ Report-PrivateTeamsChannels.ps1 -SiteUrl "https://contoso.sharepoint.com" -WhatIf

# Verbose output for troubleshooting
# .\ Report-PrivateTeamsChannels.ps1 -SiteUrl "https://contoso.sharepoint.com" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Source Credit

Sample first appeared on [Azure Automation to the Rescue – Session at Scottish Summit 2021 | CaPa Creative Ltd](https://capacreative.co.uk/2021/02/27/azure-automation-to-the-rescue-session-at-scottish-summit-2021/)

## Contributors

| Author(s) |
|-----------|
| Paul Bullock |
| [Adam Wójcik](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/report-private-teams-excel" aria-hidden="true" />

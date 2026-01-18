

# Provision Home Page to a SharePoint site 

## Summary

Copy a homepage from a source SharePoint site to a destination site with all web parts preserved and set it as the homepage using PnP PowerShell or CLI for Microsoft 365.

The script exports/copies the page from the source site and provisions it to the destination site, then sets it as the homepage.

# [PnP PowerShell](#tab/pnpps)
```powershell
   $srcUrl = Read-Host "Enter the source site url from which to copy the Home Page" #e.g.https://contoso.sharepoint.com/sites/Team1
   $destUrl = Read-Host "Enter the destination site url to which to provision the Home Page" #e.g.https://contoso.sharepoint.com/sites/testDemo
   $HomePageTemplateName = "ContosoHomePage"
try{
   $pageName = Read-Host "Enter the page name which you want to copy" ##e.g.ContosoHomePage
   Connect-PnPOnline -Url $srcUrl -interactive
   Set-location $PSScriptRoot
   Export-PnPPage -Force -Identity $pageName -Out $($HomePageTemplateName) 
}
catch{
  Write-Host -ForegroundColor Red 'Error ',':'$Error[0].ToString();
  sleep 10
} 


try{
$tempFilePath = Join-Path $PSScriptRoot $HomePageTemplateName
  Connect-PnPOnline -Url $destUrl -interactive
 Invoke-PnPSiteTemplate -Path $tempFilePath
 sleep 10
#set the page home page
 Set-PnPHomePage -RootFolderRelativeUrl SitePages/ContosoHomePage.aspx
 Write-Host "Home Page is successfully copied."
}
catch{
  Write-Host -ForegroundColor Red 'Error ',':'$Error[0].ToString();
}
 
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory, HelpMessage="Source site URL where the homepage exists")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$SourceSiteUrl,
    
    [Parameter(Mandatory, HelpMessage="Source page name (e.g., 'Home.aspx')")]
    [string]$SourcePageName,
    
    [Parameter(Mandatory, HelpMessage="Destination site URL to provision the homepage")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$DestinationSiteUrl,
    
    [Parameter(HelpMessage="Destination page name (default: same as source)")]
    [string]$DestinationPageName = "",
    
    [Parameter(HelpMessage="Output path for transcript (default: current directory)")]
    [string]$OutputPath = (Get-Location).Path,
    
    [Parameter(HelpMessage="Overwrite the target page if it already exists")]
    [switch]$Overwrite
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' first."
    }
    
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath -PathType Container)) {
            throw "Output path '$OutputPath' does not exist."
        }
    }
    
    if ([string]::IsNullOrWhiteSpace($DestinationPageName)) {
        $DestinationPageName = $SourcePageName
    }
    
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "ProvisionHomepage_Transcript_$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    $script:Summary = @{
        TotalSteps = 3
        Completed = 0
        Failed = 0
    }
    
    Write-Host "Starting homepage provisioning workflow" -ForegroundColor Cyan
    Write-Host "  Source: $SourceSiteUrl - $SourcePageName" -ForegroundColor Gray
    Write-Host "  Destination: $DestinationSiteUrl - $DestinationPageName" -ForegroundColor Gray
    Write-Host ""
}

process {
    $targetPageUrl = "$DestinationSiteUrl/SitePages/$DestinationPageName"
    
    Write-Host "[1/3] Copying page from source to destination..." -ForegroundColor Cyan
    if ($PSCmdlet.ShouldProcess($targetPageUrl, 'Copy homepage with all web parts')) {
        try {
            $copyArgs = @('spo', 'page', 'copy', '--webUrl', $SourceSiteUrl, '--sourceName', $SourcePageName, '--targetUrl', $targetPageUrl, '--output', 'json')
            if ($Overwrite) {
                $copyArgs += '--overwrite'
            }
            
            $result = m365 @copyArgs 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "CLI command failed: $result"
            }
            
            Write-Host "  SUCCESS: Page copied to $DestinationPageName" -ForegroundColor Green
            $script:Summary.Completed++
        }
        catch {
            Write-Warning "  FAILED to copy page: $_"
            $script:Summary.Failed++
        }
    } else {
        Write-Host "  WHATIF: Would copy page to $DestinationPageName" -ForegroundColor Yellow
        $script:Summary.Completed++
    }
    
    Write-Host "[2/3] Publishing copied page..." -ForegroundColor Cyan
    if ($PSCmdlet.ShouldProcess($DestinationPageName, 'Publish page')) {
        try {
            m365 spo page publish --webUrl $DestinationSiteUrl --name $DestinationPageName 2>&1 | Out-Null
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to publish page"
            }
            
            Write-Host "  SUCCESS: Page published" -ForegroundColor Green
            $script:Summary.Completed++
        }
        catch {
            Write-Warning "  FAILED to publish page: $_"
            $script:Summary.Failed++
        }
    } else {
        Write-Host "  WHATIF: Would publish page" -ForegroundColor Yellow
        $script:Summary.Completed++
    }
    
    Write-Host "[3/3] Setting page as site homepage..." -ForegroundColor Cyan
    if ($PSCmdlet.ShouldProcess($DestinationSiteUrl, 'Set as homepage')) {
        try {
            m365 spo page set --webUrl $DestinationSiteUrl --name $DestinationPageName --layoutType Home --promoteAs HomePage 2>&1 | Out-Null
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to set page as homepage"
            }
            
            Write-Host "  SUCCESS: Page set as homepage" -ForegroundColor Green
            $script:Summary.Completed++
        }
        catch {
            Write-Warning "  FAILED to set as homepage: $_"
            $script:Summary.Failed++
        }
    } else {
        Write-Host "  WHATIF: Would set page as homepage" -ForegroundColor Yellow
        $script:Summary.Completed++
    }
}

end {
    Stop-Transcript
    
    Write-Host ""
    Write-Host "=== Homepage Provisioning Summary ===" -ForegroundColor Cyan
    Write-Host "Total Steps: $($script:Summary.TotalSteps)" -ForegroundColor Gray
    
    $completedColor = if ($script:Summary.Completed -eq $script:Summary.TotalSteps) { 'Green' } else { 'Yellow' }
    Write-Host "Completed: $($script:Summary.Completed)" -ForegroundColor $completedColor
    
    $failedColor = if ($script:Summary.Failed -gt 0) { 'Red' } else { 'Green' }
    Write-Host "Failed: $($script:Summary.Failed)" -ForegroundColor $failedColor
    
    Write-Host "Transcript: $transcriptPath" -ForegroundColor Gray
    Write-Host ""
}

# Example 1: Copy homepage with WhatIf mode (safe testing)
# .\ProvisionHomepage.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/template" -SourcePageName "Home.aspx" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/newsite" -WhatIf

# Example 2: Copy homepage with custom name
# .\ProvisionHomepage.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/template" -SourcePageName "Home.aspx" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/newsite" -DestinationPageName "Welcome.aspx" -Verbose

# Example 3: Copy and overwrite existing homepage
# .\ProvisionHomepage.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/template" -SourcePageName "Home.aspx" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/newsite" -Overwrite
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Reshmee Auckloo |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-provision-homepage" aria-hidden="true" />

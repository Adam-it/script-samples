

# Hide SharePoint list from Site Contents

## Summary

If you need to hide the SharePoint list from the UI, these PowerShell scripts will hide a specific list from the site contents. This prevents users from easily accessing the list while, for example, you are still setting it up. Both PnP PowerShell and CLI for Microsoft 365 implementations are provided.
 
# [PnP PowerShell](#tab/pnpps)

```powershell

# SharePoint online site url
$siteUrl = "https://contoso.sharepoint.com/sites/SPConnect"

# Display name of SharePoint online list
$listName = "My List"

# Connect to SharePoint online site
Connect-PnPOnline -url $siteUrl -Interactive

# Hide SharePoint online list from Site Contents
Set-PnPList -Identity $listName -Hidden $true

# Disconnect SharePoint online connection
Disconnect-PnPOnline

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL (e.g., https://contoso.sharepoint.com/sites/SPConnect)")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn).*')]
   [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Display name of the SharePoint list to hide")]
   [ValidateNotNullOrEmpty()]
    [string]$ListName,

    [Parameter(Mandatory = $false, HelpMessage = "Output directory for transcript log")]
    [ValidateScript({ Test-Path -Path $_ -PathType Container -IsValid })]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyy-MM-dd-HHmmss"
    $logPath = Join-Path -Path $OutputPath -ChildPath "hide-list-transcript-$timestamp.log"
    Start-Transcript -Path $logPath

    Write-Verbose "Starting CLI for Microsoft 365 list hiding process"
    Write-Verbose "Site URL: $SiteUrl"
    Write-Verbose "List Name: $ListName"

    $script:Summary = @{
        ListsProcessed = 0
        Success = 0
        Failures = 0
    }

    Write-Host "Ensuring Microsoft 365 login..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to authenticate with Microsoft 365. Please run 'm365 login' manually and try again."
    }
    Write-Host "Successfully authenticated" -ForegroundColor Green
}

process {
    $script:Summary.ListsProcessed++

    try {
        if ($PSCmdlet.ShouldProcess($ListName, 'Hide list from Site Contents')) {
            Write-Verbose "Hiding list '$ListName' from Site Contents..."
            
            m365 spo list set --webUrl $SiteUrl --title $ListName --hidden true
            
            if ($LASTEXITCODE -ne 0) {
                throw "CLI command failed with exit code $LASTEXITCODE"
            }
            
            Write-Host "Successfully hidden list '$ListName' from Site Contents" -ForegroundColor Green
            $script:Summary.Success++
        }
        else {
            Write-Host "WhatIf: Would hide list '$ListName' from Site Contents" -ForegroundColor Yellow
        }
    }
    catch {
        Write-Warning "Failed to hide list '$ListName': $_"
        $script:Summary.Failures++
    }
}

end {
    Stop-Transcript

    Write-Host "\n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Lists Processed: $($Summary.ListsProcessed)" -ForegroundColor White
    Write-Host "Success: $($Summary.Success)" -ForegroundColor Green
    Write-Host "Failures: $($Summary.Failures)" -ForegroundColor $(if ($Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "============================\n" -ForegroundColor Cyan

    if ($Summary.Failures -gt 0) {
        Write-Host "Review the transcript log for details: $logPath" -ForegroundColor Yellow
    }
}

# Usage examples:

# Example 1: Basic usage - hide a list
# .\Hide-SPOListFromSiteContents.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/SPConnect" -ListName "My List"

# Example 2: Test with WhatIf parameter
# .\Hide-SPOListFromSiteContents.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/SPConnect" -ListName "My List" -WhatIf

# Example 3: Run with verbose output
# .\Hide-SPOListFromSiteContents.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/SPConnect" -ListName "My List" -Verbose

# Example 4: Specify custom output path for transcript
# .\Hide-SPOListFromSiteContents.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/SPConnect" -ListName "My List" -OutputPath "C:\\Logs"

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Leon Armston |
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/about) |
| Adam Wójcik (Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-hide-list-from-site-contents" aria-hidden="true" />

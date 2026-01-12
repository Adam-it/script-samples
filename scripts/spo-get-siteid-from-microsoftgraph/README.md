

# Retrieves site id from Microsoft Graph

## Summary

Retrieves a SiteId from Microsoft Graph using PnP PowerShell or CLI for Microsoft 365. This can be particularly useful when making further API calls that require the SiteId.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have access to the SharePoint Online site.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site collection")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)')]
    [string]$SiteUrl
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login status..."
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure login to CLI for Microsoft 365"
    }
}

process {
    try {
        Write-Host "Retrieving site ID for: $SiteUrl" -ForegroundColor Cyan
        Write-Verbose "Executing: m365 spo site get --url $SiteUrl --output json"
        
        $siteJson = m365 spo site get --url $SiteUrl --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve site information. CLI: $siteJson"
        }
        
        $site = $siteJson | ConvertFrom-Json
        Write-Verbose "Successfully parsed site information. Found ID: $($site.Id)"
        Write-Host "`nSite ID: " -NoNewline -ForegroundColor White
        Write-Host $site.Id -ForegroundColor Green
    }
    catch {
        Write-Error "Error retrieving site ID: $($_.Exception.Message)"
        throw
    }
}

# Usage examples:
#
# Example 1: Basic usage
# .\Get-SiteId.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Company311"
#
# Example 3: Multi-cloud (GCC High)
# .\Get-SiteId.ps1 -SiteUrl "https://contoso.sharepoint.us/sites/project"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell
$siteurl = "https://contoso.sharepoint.com/sites/Company311"
Connect-PnPOnline -url $siteurl -interactive

# Extract the domain and site name
$uri = New-Object System.Uri($siteurl)
$domain = $uri.Host
$siteName = $uri.AbsolutePath

# Construct the new URL
$RestMethodUrl = "v1.0/sites/$($domain):$($siteName)?$select=id"

$site = (Invoke-PnPGraphMethod -Url $RestMethodUrl -Method Get -ConsistencyLevelEventual)
$siteId = (($site.id) -split ",")[1]

write-host $siteId
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Retrieving SiteId from Microsoft Graph for Subsequent API Calls](https://reshmeeauckloo.com/posts/powershell_getsiteid_graph/)

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Reshmee Auckloo](https://github.com/reshmee011) |
| Ioannis Gianko|


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-siteid-from-microsoftgraph" aria-hidden="true" />

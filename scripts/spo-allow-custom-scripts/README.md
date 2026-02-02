

# Allow custom scripts in SharePoint online site

## Summary

This sample script shows how to allow use of custom scripts in SharePoint online at site level.

Scenario inspired from this blog post: [Allow use of custom scripts in SharePoint Online using PowerShell](https://ganeshsanapblogs.wordpress.com/2023/05/18/allow-use-of-custom-scripts-in-sharepoint-online-using-powershell/)

# [SPO Management Shell](#tab/spoms-ps)

```powershell

# SharePoint online admin center URL
$adminCenterUrl = "https://contoso-admin.sharepoint.com/"

# SharePoint online site URL
$siteUrl = Read-Host -Prompt "Enter your SharePoint site URL (e.g https://contoso.sharepoint.com/sites/SPConnect)"

# Connect to SharePoint online admin center
Connect-SPOService -Url $adminCenterUrl

# Allow custom scripts on SharePoint online site
Set-SPOSite $siteUrl -DenyAddAndCustomizePages 0

# Disconnect SharePoint online connection
Disconnect-SPOService

```

[!INCLUDE [More about SPO Management Shell](../../docfx/includes/MORE-SPOMS.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

# SharePoint online site URL
$siteUrl = Read-Host -Prompt "Enter your SharePoint site URL (e.g https://contoso.sharepoint.com/sites/SPConnect)"

# Connect to SharePoint online site
Connect-PnPOnline -Url $siteUrl -Interactive

# Allow custom scripts on SharePoint online site
Set-PnPSite -NoScriptSite $false

# Disconnect SharePoint online connection
Disconnect-PnPOnline

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "The URL of the SharePoint site collection to configure")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)/(sites|teams)/.+$')]
    [string]$SiteUrl
)

begin {
    Write-Host "Configuring custom script settings for SharePoint site..." -ForegroundColor Cyan
    
    # Ensure user is authenticated with CLI for Microsoft 365
    m365 login --ensure
    
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' manually."
    }
    
    Write-Verbose "Successfully authenticated with CLI for Microsoft 365"
}

process {
    if ($PSCmdlet.ShouldProcess($SiteUrl, "Enable custom scripts (set NoScriptSite to false)")) {
        try {
            Write-Verbose "Enabling custom scripts on site: $SiteUrl"
            
            # Enable custom scripts on the SharePoint site
            $output = m365 spo site set --url $SiteUrl --noScriptSite false --output json 2>&1
            
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to update site settings. CLI output: $output"
            }
            
            Write-Host "  ✓ Successfully enabled custom scripts on site" -ForegroundColor Green
            Write-Verbose "Site URL: $SiteUrl"
        }
        catch {
            Write-Error "Failed to enable custom scripts on '$SiteUrl': $_"
            throw
        }
    }
    else {
        Write-Host "WhatIf: Would enable custom scripts on $SiteUrl" -ForegroundColor Yellow
    }
}

end {
    Write-Host "`nOperation completed successfully" -ForegroundColor Cyan
    Write-Host "Note: It may take up to 15 minutes for the changes to take effect." -ForegroundColor Gray
}

# Example 1: Basic usage with mandatory parameter
# .\Enable-CustomScripts.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/SPConnect"

# Example 2: Test changes with WhatIf before applying
# .\Enable-CustomScripts.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/SPConnect" -WhatIf

# Example 3: Run with verbose output for detailed logging
# .\Enable-CustomScripts.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/SPConnect" -Verbose

# Example 4: Disable custom scripts (opposite operation)
# To disable custom scripts, you can modify the script and change --noScriptSite parameter to true
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/about) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-allow-custom-scripts" aria-hidden="true" />

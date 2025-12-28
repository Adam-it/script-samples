# Delete entire Hub site structure


## Summary

Sometimes you need to delete a hub site and all the sites associated with it. This script will do just that. It will remove all sites in the hub, unregister the hub site, and then remove the hub site itself. This sample is available in both PnP PowerShell and CLI for Microsoft 365.

![Example Screenshot](assets/example.png)

[!INCLUDE [Deletion Warning](../../docfx/includes/DELETE-WARN.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

function Remove-HubAndSites {
    param (
        [string]$hubsiteUrl,
        $adminConnection
    )

    # Get all sites in the hub
    $sitesInHub = get-pnpHubSiteChild -Identity $hubsiteUrl -Connection $adminConnection -ErrorAction Stop

    # Loop through each site and remove it
    foreach ($site in $sitesInHub) 
    {
        try {
            Remove-PnPTenantSite -Url $site -Connection $adminConnection -Force #-SkipRecycleBin
            Write-Host "Removed site: $($site)"
        } catch 
        {
            Write-Host "Failed to remove site: $($site) - $_"
            throw $_
        }
    }

    # Unregister the hub site
    try {
        Unregister-PnPHubSite -Site $hubsiteUrl -Connection $adminConnection
        Write-Host "Unregistered hub site: $hubsiteUrl"
    } 
    catch 
    {
        Write-Host "Failed to unregister hub site: $hubsiteUrl - $_"
        throw $_
    }
    # Remove the hub site itself
    try 
    {
        Remove-PnPTenantSite -Url $hubsiteUrl -Connection $adminConnection -Force #-SkipRecycleBin
        Write-Host "Removed hub site: $hubsiteUrl"
    } 
    catch 
    {
        Write-Host "Failed to remove hub site: $hubsiteUrl - $_"
        throw $_
    }
}
$adminUrl = "https://contoso-admin.sharepoint.com/"
$PnPClientId = "the PnP Client ID for your app registration"
if($null -eq $adminconn)
{
    $adminconn = Connect-PnPOnline -Url $adminUrl -Interactive -ClientId $PnPClientId -ReturnConnection
}
else
{
    Write-Host "Using existing admin connection" -ForegroundColor Yellow
}

$hubSiteUrl = "https://contoso.sharepoint.com/sites/ANL11855"
# Call the function to remove the hub site and its associated sites
Remove-HubAndSites -hubsiteUrl $hubSiteUrl -adminConnection $adminconn

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the hub site to remove along with associated sites")]
    [string]$HubSiteUrl,
    
    [Parameter(HelpMessage = "Skip recycle bin and permanently delete sites")]
    [switch]$SkipRecycleBin,
    
    [Parameter(HelpMessage = "Report only mode - show what would be removed without actually removing")]
    [switch]$ReportOnly
)

begin {
    # Verify login
    Write-Host "Verifying authentication..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to verify login status. Please run 'm365 login' first."
    }
    
    $script:Summary = @{
        AssociatedSitesFound = 0
        AssociatedSitesRemoved = 0
        AssociatedSitesFailed = 0
        HubUnregistered = $false
        HubRemoved = $false
    }
    
    if ($ReportOnly) {
        Write-Host "=== REPORT ONLY MODE - No changes will be made ===" -ForegroundColor Yellow -BackgroundColor Black
    }
}

process {
    if (-not $ReportOnly -and -not $PSCmdlet.ShouldProcess($HubSiteUrl, "Remove hub site and all associated sites")) {
        return
    }
    
    try {
        # Get hub site with associated sites
        Write-Host "Retrieving hub site information: $HubSiteUrl" -ForegroundColor Cyan
        $hubResult = m365 spo hubsite get --url $HubSiteUrl --withAssociatedSites --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve hub site information: $hubResult"
        }
        
        $hubSite = $hubResult | ConvertFrom-Json
        $associatedSites = @($hubSite.AssociatedSites)
        $script:Summary.AssociatedSitesFound = $associatedSites.Count
        
        if ($ReportOnly) {
            if ($associatedSites.Count -gt 0) {
                Write-Host "`nAssociated sites that would be removed:" -ForegroundColor Cyan
                foreach ($site in $associatedSites) {
                    Write-Host "  - $($site.Title)" -ForegroundColor White
                    Write-Host "    URL: $($site.SiteUrl)" -ForegroundColor Gray
                }
            } else {
                Write-Host "`nNo associated sites found" -ForegroundColor Gray
            }
            
            Write-Host "`nHub site that would be removed:" -ForegroundColor Cyan
            Write-Host "  - $($hubSite.Title)" -ForegroundColor White
            Write-Host "    URL: $($hubSite.SiteUrl)" -ForegroundColor Gray
            
            if ($SkipRecycleBin) {
                Write-Host "`nDeletion mode: Permanent (skip recycle bin)" -ForegroundColor Red
            } else {
                Write-Host "`nDeletion mode: Recycle bin (recoverable for 93 days)" -ForegroundColor Yellow
            }
            
            return
        }
        
        if ($associatedSites.Count -gt 0) {
            Write-Host "Found $($associatedSites.Count) associated site(s)" -ForegroundColor Yellow
            
            # Remove each associated site
            foreach ($site in $associatedSites) {
                try {
                    Write-Verbose "Removing site: $($site.SiteUrl)"
                    
                    if ($SkipRecycleBin) {
                        $removeResult = m365 spo site remove --url $site.SiteUrl --skipRecycleBin --force 2>&1
                    } else {
                        $removeResult = m365 spo site remove --url $site.SiteUrl --force 2>&1
                    }
                    
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to remove site '$($site.SiteUrl)': $removeResult"
                        $script:Summary.AssociatedSitesFailed++
                        continue
                    }
                    
                    $script:Summary.AssociatedSitesRemoved++
                    Write-Verbose "Successfully removed: $($site.SiteUrl)"
                } catch {
                    Write-Warning "Error removing site '$($site.SiteUrl)': $_"
                    $script:Summary.AssociatedSitesFailed++
                    continue
                }
            }
        } else {
            Write-Host "No associated sites found" -ForegroundColor Gray
        }
        
        # Unregister hub site
        Write-Host "Unregistering hub site: $HubSiteUrl" -ForegroundColor Cyan
        $unregisterResult = m365 spo hubsite unregister --url $HubSiteUrl --force 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to unregister hub site: $unregisterResult"
        }
        $script:Summary.HubUnregistered = $true
        Write-Verbose "Successfully unregistered hub site"
        
        # Remove the hub site itself
        Write-Host "Removing hub site: $HubSiteUrl" -ForegroundColor Cyan
        if ($SkipRecycleBin) {
            $hubRemoveResult = m365 spo site remove --url $HubSiteUrl --skipRecycleBin --force 2>&1
        } else {
            $hubRemoveResult = m365 spo site remove --url $HubSiteUrl --force 2>&1
        }
        
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to remove hub site: $hubRemoveResult"
        }
        $script:Summary.HubRemoved = $true
        Write-Verbose "Successfully removed hub site"
        
    } catch {
        Write-Error "Critical error during hub site removal: $_"
        throw
    }
}

end {
    if ($ReportOnly) {
        Write-Host "`n" -NoNewline
        Write-Host "=== Report Summary ===" -ForegroundColor Cyan
        Write-Host "Hub Site URL: " -NoNewline; Write-Host $HubSiteUrl -ForegroundColor Yellow
        Write-Host "Associated Sites: " -NoNewline; Write-Host $script:Summary.AssociatedSitesFound -ForegroundColor White
        Write-Host "Total Sites to Remove: " -NoNewline; Write-Host ($script:Summary.AssociatedSitesFound + 1) -ForegroundColor White
        Write-Host "`nTo proceed with removal, run the script without -ReportOnly switch" -ForegroundColor Yellow
        return
    }
    
    Write-Host "`n" -NoNewline
    Write-Host "=== Hub Site Removal Summary ===" -ForegroundColor Cyan
    Write-Host "Hub Site URL: " -NoNewline; Write-Host $HubSiteUrl -ForegroundColor Yellow
    Write-Host "Associated Sites Found: " -NoNewline; Write-Host $script:Summary.AssociatedSitesFound -ForegroundColor White
    Write-Host "Associated Sites Removed: " -NoNewline; Write-Host $script:Summary.AssociatedSitesRemoved -ForegroundColor Green
    
    if ($script:Summary.AssociatedSitesFailed -gt 0) {
        Write-Host "Associated Sites Failed: " -NoNewline; Write-Host $script:Summary.AssociatedSitesFailed -ForegroundColor Red
    }
    
    Write-Host "Hub Unregistered: " -NoNewline
    if ($script:Summary.HubUnregistered) {
        Write-Host "Yes" -ForegroundColor Green
    } else {
        Write-Host "No" -ForegroundColor Red
    }
    
    Write-Host "Hub Site Removed: " -NoNewline
    if ($script:Summary.HubRemoved) {
        Write-Host "Yes" -ForegroundColor Green
    } else {
        Write-Host "No" -ForegroundColor Red
    }
    
    if (-not $SkipRecycleBin -and ($script:Summary.AssociatedSitesRemoved -gt 0 -or $script:Summary.HubRemoved)) {
        Write-Host "`nNote: Sites moved to recycle bin. They can be restored within 93 days." -ForegroundColor Yellow
    }
}

# Example 1: Report mode - preview what would be removed
# .\Remove-HubAndSites.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/MarketingHub" -ReportOnly

# Example 2: Remove hub and associated sites (move to recycle bin)
# .\Remove-HubAndSites.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/MarketingHub"

# Example 3: Permanently delete hub and sites (skip recycle bin)
# .\Remove-HubAndSites.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/MarketingHub" -SkipRecycleBin

# Example 4: Preview what would be deleted with WhatIf
# .\Remove-HubAndSites.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/MarketingHub" -WhatIf

# Example 5: Verbose output for detailed progress
# .\Remove-HubAndSites.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/MarketingHub" -Verbose

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-delete-hub-and-sites" aria-hidden="true" />

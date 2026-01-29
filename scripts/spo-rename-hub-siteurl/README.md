# Renaming a SharePoint Hub Site URL

## Summary

Renaming a SharePoint Hub Site URL or title is not a straightforward process. A hub site can not be renamed directly. Instead, the hub site needs to be unregistered before performing the rename, and then re-register it as a hub. This ensures the integrity of the hub structure and keeps associated sites intact. This sample demonstrates the workflow using both CLI for Microsoft 365 and PnP PowerShell.

---

## 🛠️ What This Script Does

This PowerShell script walks through:

✅ Unregistering the hub site

✏️ Renaming the site URL and title

🗑️ Deleting the redirect site automatically created after renaming

🔁 Re-registering the site as a hub

🔍 Verifying that child sites remain associated

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have access to the SharePoint Online site.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "Current URL of the hub site to rename")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$CurrentSiteUrl,
    
    [Parameter(Mandatory = $true, HelpMessage = "New URL for the hub site")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$NewSiteUrl,
    
    [Parameter(Mandatory = $false, HelpMessage = "Optional new title for the hub site")]
    [string]$NewTitle
)

begin {
    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure
    
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365. Please run 'm365 login' and try again."
    }
    
    Write-Host "Retrieving hub site information from $CurrentSiteUrl..." -ForegroundColor Cyan
    $hubSiteJson = m365 spo hubsite get --url $CurrentSiteUrl --withAssociatedSites --output json 2>&1
    
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve hub site information. Please verify the site URL and ensure it is a hub site."
    }
    
    $script:HubSite = $hubSiteJson | ConvertFrom-Json
    
    if (-not $script:HubSite.ID) {
        throw "The site at '$CurrentSiteUrl' is not registered as a hub site."
    }
    
    $script:InitialChildSites = @($script:HubSite.AssociatedSites)
    $script:InitialChildCount = $script:InitialChildSites.Count
    
    Write-Host "Hub site found: '$($script:HubSite.Title)' with $script:InitialChildCount associated site(s)" -ForegroundColor Green
    
    $script:Summary = @{
        OldUrl = $CurrentSiteUrl
        NewUrl = $NewSiteUrl
        OldTitle = $script:HubSite.Title
        NewTitle = if ($NewTitle) { $NewTitle } else { $script:HubSite.Title }
        SiteId = $script:HubSite.ID
        InitialChildCount = $script:InitialChildCount
    }
    
    $script:TranscriptPath = "RenameHubSite-$(Get-Date -Format "yyyyMMdd-HHmmss").log"
    Start-Transcript -Path $script:TranscriptPath | Out-Null
    Write-Verbose "Transcript started at: $script:TranscriptPath"
}

process {
    try {
        Write-Host "`n========================================" -ForegroundColor Yellow
        Write-Host "Step 1: Unregistering hub site" -ForegroundColor Yellow
        Write-Host "========================================" -ForegroundColor Yellow
        
        if ($PSCmdlet.ShouldProcess($CurrentSiteUrl, 'Unregister hub site')) {
            Write-Verbose "Executing: m365 spo hubsite unregister --url $CurrentSiteUrl --force"
            m365 spo hubsite unregister --url $CurrentSiteUrl --force
            
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to unregister hub site at '$CurrentSiteUrl'."
            }
            
            Write-Host "Hub site unregistered successfully" -ForegroundColor Green
        }
        
        Write-Host "`n========================================" -ForegroundColor Yellow
        Write-Host "Step 2: Renaming site URL" -ForegroundColor Yellow
        Write-Host "========================================" -ForegroundColor Yellow
        
        $renameAction = if ($NewTitle) { "Rename to $NewSiteUrl with title '$NewTitle'" } else { "Rename to $NewSiteUrl" }
        
        if ($PSCmdlet.ShouldProcess($CurrentSiteUrl, $renameAction)) {
            $renameCommand = "m365 spo site rename --url '$CurrentSiteUrl' --newUrl '$NewSiteUrl'"
            if ($NewTitle) {
                $renameCommand += " --newTitle '$NewTitle'"
            }
            $renameCommand += " --wait --output json"
            
            Write-Verbose "Executing: $renameCommand"
            
            $renameJson = if ($NewTitle) {
                m365 spo site rename --url $CurrentSiteUrl --newUrl $NewSiteUrl --newTitle $NewTitle --wait --output json
            } else {
                m365 spo site rename --url $CurrentSiteUrl --newUrl $NewSiteUrl --wait --output json
            }
            
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to rename site from '$CurrentSiteUrl' to '$NewSiteUrl'."
            }
            
            $renameResult = $renameJson | ConvertFrom-Json
            
            if ($renameResult.ErrorCode -and $renameResult.ErrorCode -ne 0) {
                throw "Site rename completed with error code: $($renameResult.ErrorCode)"
            }
            
            Write-Host "Site renamed successfully" -ForegroundColor Green
            Write-Verbose "Rename job ID: $($renameResult.JobId)"
        }
        
        Write-Host "`n========================================" -ForegroundColor Yellow
        Write-Host "Step 3: Removing redirect site" -ForegroundColor Yellow
        Write-Host "========================================" -ForegroundColor Yellow
        
        if ($PSCmdlet.ShouldProcess($CurrentSiteUrl, 'Remove redirect site permanently')) {
            Write-Host "Waiting 10 seconds for redirect site creation..." -ForegroundColor Cyan
            Start-Sleep -Seconds 10
            
            Write-Verbose "Executing: m365 spo site remove --url $CurrentSiteUrl --skipRecycleBin --force"
            m365 spo site remove --url $CurrentSiteUrl --skipRecycleBin --force
            
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to remove redirect site at '$CurrentSiteUrl'. It may not exist or may require manual removal."
            } else {
                Write-Host "Redirect site removed successfully" -ForegroundColor Green
            }
        }
        
        Write-Host "`n========================================" -ForegroundColor Yellow
        Write-Host "Step 4: Re-registering as hub site" -ForegroundColor Yellow
        Write-Host "========================================" -ForegroundColor Yellow
        
        if ($PSCmdlet.ShouldProcess($NewSiteUrl, 'Re-register as hub site')) {
            Write-Verbose "Executing: m365 spo hubsite register --siteUrl $NewSiteUrl --output json"
            $registerJson = m365 spo hubsite register --siteUrl $NewSiteUrl --output json
            
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to re-register hub site at '$NewSiteUrl'."
            }
            
            Write-Host "Hub site re-registered successfully" -ForegroundColor Green
        }
        
        Write-Host "`n========================================" -ForegroundColor Yellow
        Write-Host "Step 5: Verifying child site associations" -ForegroundColor Yellow
        Write-Host "========================================" -ForegroundColor Yellow
        
        Write-Verbose "Retrieving updated hub site information..."
        $updatedHubJson = m365 spo hubsite get --url $NewSiteUrl --withAssociatedSites --output json
        
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to retrieve updated hub site information for verification."
            $script:Summary.FinalChildCount = "Unknown"
            $script:Summary.ChildSitesMatch = "Unknown"
        } else {
            $updatedHub = $updatedHubJson | ConvertFrom-Json
            $finalChildSites = @($updatedHub.AssociatedSites)
            $script:Summary.FinalChildCount = $finalChildSites.Count
            
            if ($script:InitialChildCount -eq 0) {
                $script:Summary.ChildSitesMatch = "Yes (no child sites)"
                Write-Host "No child sites were associated - verification complete" -ForegroundColor Green
            } else {
                $comparison = Compare-Object -ReferenceObject ($script:InitialChildSites | Select-Object -ExpandProperty SiteUrl) -DifferenceObject ($finalChildSites | Select-Object -ExpandProperty SiteUrl)
                
                if ($comparison) {
                    $script:Summary.ChildSitesMatch = "No - changes detected"
                    Write-Host "WARNING: Child site associations have changed!" -ForegroundColor Red
                    Write-Host "Initial count: $script:InitialChildCount, Final count: $($finalChildSites.Count)" -ForegroundColor Yellow
                } else {
                    $script:Summary.ChildSitesMatch = "Yes"
                    Write-Host "All $($finalChildSites.Count) child site(s) remain associated" -ForegroundColor Green
                }
            }
        }
    }
    catch {
        Write-Host "`n========================================" -ForegroundColor Red
        Write-Host "ERROR: Hub site rename failed" -ForegroundColor Red
        Write-Host "========================================" -ForegroundColor Red
        Write-Host $_.Exception.Message -ForegroundColor Red
        throw
    }
}

end {
    Stop-Transcript | Out-Null
    
    
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Hub Site Rename Complete" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Old URL:      " -NoNewline -ForegroundColor Gray
    Write-Host $script:Summary.OldUrl -ForegroundColor White
    Write-Host "New URL:      " -NoNewline -ForegroundColor Gray
    Write-Host $script:Summary.NewUrl -ForegroundColor Green
    Write-Host "Old Title:    " -NoNewline -ForegroundColor Gray
    Write-Host $script:Summary.OldTitle -ForegroundColor White
    Write-Host "New Title:    " -NoNewline -ForegroundColor Gray
    Write-Host $script:Summary.NewTitle -ForegroundColor Green
    Write-Host "Child Sites:  " -NoNewline -ForegroundColor Gray
    
    $childColor = if ($script:Summary.ChildSitesMatch -like "*Yes*") { "Green" } elseif ($script:Summary.ChildSitesMatch -like "*Unknown*") { "Yellow" } else { "Red" }
    Write-Host "$($script:Summary.FinalChildCount) associated (" -NoNewline -ForegroundColor White
    Write-Host $script:Summary.ChildSitesMatch -NoNewline -ForegroundColor $childColor
    Write-Host ")" -ForegroundColor White
    
    Write-Host "Transcript:   " -NoNewline -ForegroundColor Gray
    Write-Host $script:TranscriptPath -ForegroundColor Cyan
}

# Example 1: Rename hub site URL with WhatIf preview
# .\Rename-HubSite.ps1 -CurrentSiteUrl "https://contoso.sharepoint.com/sites/TestHR_Hub" -NewSiteUrl "https://contoso.sharepoint.com/sites/TestHR_Hub_Updated" -WhatIf

# Example 2: Rename hub site with new title
# .\Rename-HubSite.ps1 -CurrentSiteUrl "https://contoso.sharepoint.com/sites/TestHR_Hub" -NewSiteUrl "https://contoso.sharepoint.com/sites/TestHR_Hub_Updated" -NewTitle "HR Hub - Updated"


# Example 3: Rename with verbose output
# .\Rename-HubSite.ps1 -CurrentSiteUrl "https://contoso.sharepoint.com/sites/TestHR_Hub" -NewSiteUrl "https://contoso.sharepoint.com/sites/TestHR_Hub_Updated" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
param (
    [Parameter(Mandatory = $true)]
    [string]$domain = "contoso",

    [Parameter(Mandatory = $true)]
    [string]$currentSiteUrl = "https://contoso.sharepoint.com/sites/TestHR_Hub",

    [Parameter(Mandatory = $true)]
    [string]$updatedSiteUrl = "https://contoso.sharepoint.com/sites/TestHR_Hub_Updated"
)

$adminSiteURL = "https://$domain-Admin.SharePoint.com"
Connect-PnPOnline -Url $adminSiteURL

try {
    $siteId = (Get-PnPTenantSite -Identity $currentSiteUrl).SiteId
    $hubSite = Get-PnPHubSite -Identity $siteId.Guid -ErrorAction SilentlyContinue

    if (-not $hubSite) {
        Write-Host "The site is not a hub site or does not exist."
    }
    else {
        $childSites = Get-PnPHubSiteChild -Identity $siteId.Guid -ErrorAction SilentlyContinue
        Unregister-PnPHubSite -Site $siteId.Guid
    }

    Rename-PnPTenantSite -Identity $currentSiteUrl -NewSiteUrl $updatedSiteUrl -NewSiteTitle 'Test HR Hub' | Out-Null

    # Wait for the rename to complete
    $updatehubSite = Get-PnPTenantSite -Identity $updatedSiteUrl -ErrorAction SilentlyContinue
    while(-not $updatehubSite) {
        Start-Sleep -Seconds 30
        $updatehubSite = Get-PnPTenantSite -Identity $updatedSiteUrl -ErrorAction SilentlyContinue
    }

    # Remove the redirect site created by the rename
    Remove-PnPTenantSite -Url $currentSiteUrl -Force

    # Register the site as a hub site again
    Register-PnPHubSite -Site $updatedSiteUrl | Out-Null

    # Check if child sites are still associated
    $childSites1 = Get-PnPHubSiteChild -Identity $siteId.Guid -ErrorAction SilentlyContinue
    $result = Compare-Object -ReferenceObject $childSites1 -DifferenceObject $childSites
    if ($result) {
        Write-Host "Child sites have been updated after renaming the hub site."
    } else {
        Write-Host "No changes in child sites after renaming the hub site."
    }
}
catch {
    Write-Host "An error occurred while renaming the hub site: $_"
}
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [How to Safely Rename a SharePoint Hub Site URL with PnP PowerShell](https://reshmeeauckloo.com/posts/powershell-sharepoint-rename-hubsite/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-rename-hub-siteurl" aria-hidden="true" />

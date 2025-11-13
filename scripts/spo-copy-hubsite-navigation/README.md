

# Copy a hub navigation from a source site to any desired target hub site

## Summary

This script copies a hub navigation from any hub site (source) to another hub site (target). Use this script to create a consistent hub navigation for all your sites in SharePoint Online.

Based on the navigation structure of any hub site of your choice – e.g. the hub navigation of your home site, which serves as a template - any desired hub navigation within your SharePoint Online site architecture can be created identically.

> [!NOTE]
> The deployment process is idempotent; each navigation is only deployed once and replaced in the target hub site when it is deployed again. You can start the copying process as often as you like without expecting any side effects!


# [PnP PowerShell](#tab/pnpps)

```powershell

Function Copy-Hubnavigation {
  [cmdletbinding()]
  param(
    [Parameter(
      Mandatory = $true
    )][string]$SourceSiteRelativeUrl,
    [Parameter(
      Mandatory = $true
    )][string]$DestinationSiteRelativeUrl
  )

  Function Get-ToplevelHubnavigation([PnP.PowerShell.Commands.Base.PnPConnection]$sourceSiteConn) {
    try {
      $hubsiteNavigation = Get-PnPNavigationNode -Location TopNavigationBar -Connection $sourceSiteConn
      return $hubsiteNavigation
    }
    catch {
      Write-Host " ✘ failed: $($_)" -ForegroundColor Red
    }
  }

  Function New-HubnavigationElement([Object]$naviItem, [Object]$parentItem, [PnP.PowerShell.Commands.Base.PnPConnection]$destSiteConn, [PnP.PowerShell.Commands.Base.PnPConnection]$sourceSiteConn) {
    # construct path based on given relative url of item
    $naviItem.Url = $naviItem.Url -eq "http://linkless.header/" ? "http://linkless.header/" : ($naviItem.Url.StartsWith("https://") ? $naviItem.Url : "$($naviItem.Context.Url)$($naviItem.Url.TrimStart('/'))")
    if ($null -ne $parentItem) {
      $node = Add-PnPNavigationNode -Location TopNavigationBar -Title $naviItem.Title -Url $naviItem.Url -Parent $parentItem.Id -Connection $destSiteConn
    }
    else { 
      $node = Add-PnPNavigationNode -Location TopNavigationBar -Title $naviItem.Title -Url $naviItem.Url -Connection $destSiteConn 
    }
    
    # handle child nodes (recursively)
    if ($null -ne $naviItem.Children) {
      foreach ($childNaviItem in $naviItem.Children) {
        # get the details about the node:
        $childNaviItem = Get-PnPNavigationNode -Id $childNaviItem.Id -Connection $connHubsiteSource
        New-HubnavigationElement -naviItem $childNaviItem -parentItem $node -destSiteConn $destSiteConn
      }
    }
  }

  if ($null -eq $SourceSiteRelativeUrl) { throw "No Source Hubsite Url provided" }
  if ($null -eq $DestinationSiteRelativeUrl) { throw "No Destination Hubsite Url provided" }

  $connAdmin = Get-PnPConnection
  $spoBaseUrl = $connAdmin.Url.Replace('-admin', '')

  $spoUrlSource = "$($spoBaseUrl)$($SourceSiteRelativeUrl)"
  $spoUrlDestination = "$($spoBaseUrl)$($DestinationSiteRelativeUrl)"

  $connHubsiteSource = Connect-PnPOnline -Url $spoUrlSource -ReturnConnection -Interactive
  $connHubSiteDest = Connect-PnPOnline -Url $spoUrlDestination -ReturnConnection -Interactive

  Write-Host "⭐️ Site '$($DestinationSiteRelativeUrl)';"
  try {
    # Delete all existing nodes:
    Remove-PnPNavigationNode -Force -All -Connection $connHubSiteDest

    Write-Host "⎿  Copying consistent hub navigation from '$($connHubsiteSource.Url)': " -NoNewline
    $navigation = Get-ToplevelHubnavigation -sourceSiteConn $connHubsiteSource
    foreach ($naviItem in $navigation) {
      # get the details about the node:
      $naviItem = Get-PnPNavigationNode -Id $naviItem.Id -Connection $connHubsiteSource 
      New-HubnavigationElement -naviItem $naviItem -parentItem $null -destSiteConn $connHubSiteDest -sourceSiteConn $connHubsiteSource
    }
    Write-Host " ✔︎ Done" -ForegroundColor DarkGreen
  }
  catch {
    Write-Host " ✘ failed: $($_)" -ForegroundColor Red
  }
  finally {
    $connHubsiteSource = $null
    $connHubSiteDest = $null
    $connAdmin = $null
  }

}

# First connect to admin site of your tenant; make sure you are an SPO Admin:
Connect-PnPOnline "https://[tenant]-admin.sharepoint.com" -Interactive

# Copy hub navigation from the soure hub site (here "/") to the destination hub site (here "/sites/LearningHub"):
Copy-Hubnavigation -SourceSiteRelativeUrl "/" -DestinationSiteRelativeUrl "/sites/LearningHub"
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
# .\Copy-HubNavigation.ps1 -SourceHubUrl "https://contoso.sharepoint.com/sites/Home" -DestinationHubUrl "https://contoso.sharepoint.com/sites/Learning" -WhatIf
[CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'Medium')]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Absolute URL of the source hub site whose navigation will be copied.")]
    [ValidatePattern('^https://')]
    [string]$SourceHubUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Absolute URL of the destination hub site that will receive the navigation.")]
    [ValidatePattern('^https://')]
    [string]$DestinationHubUrl
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 session."
    m365 login --ensure --output json | Out-Null

    Write-Host "Fetching source hub navigation from '$SourceHubUrl'" -ForegroundColor Cyan
    $sourceNavigationJson = m365 spo hubsite navigation get --webUrl $SourceHubUrl --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve hub navigation. CLI output: $sourceNavigationJson"
    }

    $Script:SourceNavigation = $sourceNavigationJson | ConvertFrom-Json
    if (-not $Script:SourceNavigation) {
        throw "Source navigation is empty. No items to copy."
    }
}

process {}

end {
    Write-Host "Preparing navigation payload for '$DestinationHubUrl'" -ForegroundColor Cyan
    $navigationPayload = $Script:SourceNavigation | ConvertTo-Json -Depth 10 -Compress

    if (-not $PSCmdlet.ShouldProcess($DestinationHubUrl, 'Replace hub navigation')) {
        Write-Host "WhatIf: navigation not updated on '$DestinationHubUrl'" -ForegroundColor Yellow
        return
    }

    $setOutput = m365 spo hubsite navigation set --webUrl $DestinationHubUrl --navigation $navigationPayload --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to apply navigation to '$DestinationHubUrl'. CLI output: $setOutput"
    }

    Write-Host "Hub navigation successfully copied to '$DestinationHubUrl'" -ForegroundColor Green
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Source Credit

Sample taken from [https://github.com/tmaestrini/easyProvisioning/](https://github.com/tmaestrini/easyProvisioning)

## Contributors

| Author(s) |
|-----------|
| [Tobias Maestrini](https://github.com/tmaestrini)|
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-copy-hubsite-navigation" aria-hidden="true" />

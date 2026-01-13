

# Get (or export) an existing site structure in a SharePoint Online tenant

## Summary
  The `Get-SiteStructure` function is a PowerShell function that could be part of a larger script or module. 
  Its purpose is to provide a function that retrieves an existing structure of a SharePoint Online (SPO) tenant
  and its content for further processing.

  The function takes in several parameters:
  * `$RootSiteUrl`: The URL of the root site to start with.
  * `$WithSiteContent`: A switch parameter that determines whether the site content (libraries) should be included.
  * `$AsObject`: A switch parameter that determines whether the result will be returned as an object to be processed later.

  The main function consists of several sub-functions define the analysis or export process:
  1. initialize some variables,
  1. analyze whether to start with a home site or hub site,
  1. get informations about the selected root site,
  1. get all assigned sites (and hubs),
  1. and optionally retrieve also site content (document libraries and lists). 
  
  Finally, `Get-SiteStructure` returns the site structure, optionally as an object that can be processed later:
  * `Tenant`: The tenant's name
  * `Version`: Timestamp of the execution
  * `SharePoint`: High-level inforamtion about the tenant (for now, only the `TenantId`)
  * `Structure`: The retrieved structure, consisting of sites and site content (optional)

> [!NOTE]
> Both scripts require SharePoint administrator rights.

## Usage

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
# Basic usage - analyze hub structure starting from a hub site
.\Get-SiteStructure.ps1 -RootSiteUrl "https://contoso.sharepoint.com/sites/hub"

# Include document libraries in the analysis
.\Get-SiteStructure.ps1 -RootSiteUrl "https://contoso.sharepoint.com/sites/hub" -WithSiteContent

# Return result as object for further processing
$result = .\Get-SiteStructure.ps1 -RootSiteUrl "https://contoso.sharepoint.com/sites/hub" -WithSiteContent -AsObject

# Export to JSON file
$result = .\Get-SiteStructure.ps1 -RootSiteUrl "https://contoso.sharepoint.com/sites/hub" -WithSiteContent -AsObject
$result | ConvertTo-Json -Depth 10 | Out-File "hub-structure.json"

# Run with verbose logging for troubleshooting
.\Get-SiteStructure.ps1 -RootSiteUrl "https://contoso.sharepoint.com/sites/hub" -Verbose
```

> [!NOTE]
> - The script uses `m365 login --ensure` which prompts for authentication if not already logged in
> - Login session persists across script runs (no need to re-authenticate)
> - Transcript log is automatically created: `Get-SiteStructure_yyyyMMdd_HHmmss.log`
> - The script supports multi-home site scenarios (audience-targeted home sites)

# [PnP PowerShell](#tab/pnpps)

```powershell
# First, connect to SPO admin center
$adminUrl = "your SPO tenant admin url"
Connect-PnPOnline -Url $adminUrl -Interactive

# Basic usage
Get-SiteStructure -RootSiteUrl "https://<yourtenant>.sharepoint.com/sites/<your(hub)site>" -WithSiteContent

# Store output as object
$result = Get-SiteStructure -RootSiteUrl "https://<yourtenant>.sharepoint.com/sites/<your(hub)site>" -WithSiteContent -AsObject

# Export to JSON
$result | ConvertTo-Json -Depth 10 | Out-File "hub-structure.json"
```

> [!NOTE]
> - Requires active connection to SPO admin center via `Connect-PnPOnline`
> - Must reconnect for each site being analyzed (N+1 connections)

***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding()]
param(
    [Parameter(Mandatory = $true, HelpMessage = "The root site URL to start with (must be a home site or hub site)")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)(/.*)?$')]
    [string]$RootSiteUrl,
    
    [Parameter(Mandatory = $false, HelpMessage = "Include document libraries in the output")]
    [switch]$WithSiteContent,
    
    [Parameter(Mandatory = $false, HelpMessage = "Return result as an object instead of displaying it")]
    [switch]$AsObject
)

begin {
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure CLI login. Please run 'm365 login' manually."
    }
    
    $script:Structure = @()
    $script:ProcessedSites = @{}
    $script:Summary = @{
        HubsProcessed  = 0
        SitesProcessed = 0
        Failures       = 0
    }
    
    $script:ExcludedLibraries = @(
        'Form Templates', 'FormServerTemplates', 'Site Assets', 'SiteAssets',
        'Site Pages', 'SitePages', 'Style Library', 'Style_x0020_Library',
        'Images', 'Pages', 'Preservation Hold Library', 'Cache Profiles',
        'Content type publishing error log', 'Converted Forms', 'Device Channels',
        'Drop Off Library', 'Long Running Operation Status', 'Quick Deploy Items',
        'Relationships List', 'Solution Gallery', 'TaxonomyHiddenList',
        'Theme Gallery', 'Translation Packages', 'Variation Labels', 'Web Part Gallery'
    )
    
    $TenantName = if ($RootSiteUrl -match 'https://([^.]+)\\.sharepoint') { $Matches[1] } else { 'Unknown' }
    
    $transcriptPath = "$((Get-Location).Path)/Get-SiteStructure_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    Start-Transcript -Path $transcriptPath -Append | Out-Null
    
    Write-Verbose "Starting site structure analysis from: $RootSiteUrl"
    
    function Get-SiteType {
        param([string]$WebTemplate)
        
        switch ($WebTemplate) {
            'SITEPAGEPUBLISHING#0' { return 'Communication' }
            'GROUP#0' { return 'Team' }
            'STS#3' { return 'SPOTeam' }
            default { return 'Other' }
        }
    }
    
    function Get-SiteContent {
        param([string]$SiteUrl)
        
        try {
            Write-Verbose "  Retrieving lists from: $SiteUrl"
            $listsJson = m365 spo list list --webUrl $SiteUrl --query "[?Hidden == \\`false\\` && BaseType == \\`1\\`]" --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve lists from $SiteUrl"
                return @()
            }
            
            $lists = @($listsJson | ConvertFrom-Json)
            $filteredLists = $lists | Where-Object { $_.Title -notin $script:ExcludedLibraries }
            
            return @($filteredLists | ForEach-Object {
                    @{
                        Id                     = $_.Id
                        Title                  = $_.Title
                        ServerRelativeUrl      = $_.RootFolder.ServerRelativeUrl
                        ItemCount              = $_.ItemCount
                        DocumentTemplateUrl    = $_.DocumentTemplateUrl
                        DefaultViewUrl         = $_.DefaultViewUrl
                        HasUniqueRoleAssignments = $_.HasUniqueRoleAssignments
                    }
                })
        }
        catch {
            Write-Warning "Error retrieving content from $SiteUrl : $_"
            return @()
        }
    }
    
    function Get-SiteInfo {
        param(
            [string]$SiteUrl,
            [string]$ConnectedHubUrl = $null
        )
        
        if ($script:ProcessedSites.ContainsKey($SiteUrl)) {
            Write-Verbose "  Site already processed: $SiteUrl"
            return $null
        }
        
        try {
            Write-Verbose "  Processing site: $SiteUrl"
            $siteJson = m365 spo site get --url $SiteUrl --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to get site info for $SiteUrl . CLI: $siteJson"
                $script:Summary.Failures++
                return $null
            }
            
            $site = $siteJson | ConvertFrom-Json
            
            $webJson = m365 spo web get --url $SiteUrl --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to get web info for $SiteUrl . CLI: $webJson"
                $script:Summary.Failures++
                return $null
            }
            
            $web = $webJson | ConvertFrom-Json
            
            $script:ProcessedSites[$SiteUrl] = $true
            $script:Summary.SitesProcessed++
            
            $siteStructure = @{
                Id               = $site.Id
                Title            = $web.Title
                Url              = $site.Url
                Type             = Get-SiteType -WebTemplate $web.WebTemplate
                IsHubSite        = $site.IsHubSite
                ConnectedHubUrl  = $ConnectedHubUrl
            }
            
            if ($WithSiteContent) {
                $siteStructure['Content'] = Get-SiteContent -SiteUrl $SiteUrl
            }
            
            return $siteStructure
        }
        catch {
            Write-Warning "Error processing site $SiteUrl : $_"
            $script:Summary.Failures++
            return $null
        }
    }
    
    function Process-HubSite {
        param(
            [string]$HubUrl,
            [object]$HubData = $null
        )
        
        try {
            if (-not $HubData) {
                Write-Verbose "Retrieving hub site details: $HubUrl"
                $hubJson = m365 spo hubsite get --url $HubUrl --withAssociatedSites --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to get hub site $HubUrl . CLI: $hubJson"
                    $script:Summary.Failures++
                    return $null
                }
                $HubData = $hubJson | ConvertFrom-Json
            }
            
            $script:Summary.HubsProcessed++
            
            $hubSiteInfo = Get-SiteInfo -SiteUrl $HubData.SiteUrl
            if (-not $hubSiteInfo) { return $null }
            
            $childSites = @()
            if ($HubData.AssociatedSites -and $HubData.AssociatedSites.Count -gt 0) {
                Write-Verbose "Processing $($HubData.AssociatedSites.Count) associated sites for hub: $($HubData.SiteUrl)"
                
                foreach ($associatedSite in $HubData.AssociatedSites) {
                    try {
                        $siteJson = m365 spo site get --url $associatedSite.SiteUrl --output json 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to check if $($associatedSite.SiteUrl) is a hub"
                            continue
                        }
                        
                        $siteInfo = $siteJson | ConvertFrom-Json
                        
                        if ($siteInfo.IsHubSite) {
                            $childHub = Process-HubSite -HubUrl $associatedSite.SiteUrl
                            if ($childHub) {
                                $childSites += $childHub
                            }
                        }
                        else {
                            $siteStructure = Get-SiteInfo -SiteUrl $associatedSite.SiteUrl -ConnectedHubUrl $HubData.SiteUrl
                            if ($siteStructure) {
                                $childSites += $siteStructure
                            }
                        }
                    }
                    catch {
                        Write-Warning "Error processing associated site $($associatedSite.SiteUrl) : $_"
                        $script:Summary.Failures++
                        continue
                    }
                }
            }
            
            if ($childSites.Count -gt 0) {
                $hubSiteInfo['Sites'] = $childSites
            }
            
            return $hubSiteInfo
        }
        catch {
            Write-Warning "Error processing hub site $HubUrl : $_"
            $script:Summary.Failures++
            return $null
        }
    }
}

process {
    Write-Host "Checking if root site is a home site..." -ForegroundColor Cyan
    $homeSitesJson = m365 spo homesite list --output json 2>&1
    
    $isHomeSite = $false
    if ($LASTEXITCODE -eq 0) {
        $homeSites = @($homeSitesJson | ConvertFrom-Json)
        $matchingHomeSite = $homeSites | Where-Object { $_.Url -eq $RootSiteUrl }
        if ($matchingHomeSite) {
            $isHomeSite = $true
            Write-Host "  Root site is a home site: $($matchingHomeSite.Title)" -ForegroundColor Green
        }
    }
    
    if ($isHomeSite) {
        $siteJson = m365 spo site get --url $RootSiteUrl --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to get home site info. CLI: $siteJson"
        }
        
        $siteInfo = $siteJson | ConvertFrom-Json
        
        if ($siteInfo.IsHubSite) {
            Write-Host "Home site is also a hub site, processing as hub..." -ForegroundColor Cyan
            $hubStructure = Process-HubSite -HubUrl $RootSiteUrl
            if ($hubStructure) {
                $script:Structure += $hubStructure
            }
        }
        else {
            Write-Host "Home site is not a hub, processing as regular site..." -ForegroundColor Cyan
            $siteStructure = Get-SiteInfo -SiteUrl $RootSiteUrl
            if ($siteStructure) {
                $script:Structure += $siteStructure
            }
        }
    }
    else {
        Write-Host "Root site is not a home site, checking if it's a hub site..." -ForegroundColor Cyan
        
        $hubJson = m365 spo hubsite get --url $RootSiteUrl --withAssociatedSites --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Root site is neither a home site nor a hub site. Cannot proceed. CLI: $hubJson"
        }
        
        $hubInfo = $hubJson | ConvertFrom-Json
        Write-Host "  Root site is a hub site: $($hubInfo.Title)" -ForegroundColor Green
        
        $hubStructure = Process-HubSite -HubUrl $RootSiteUrl -HubData $hubInfo
        if ($hubStructure) {
            $script:Structure += $hubStructure
        }
    }
}

end {
    $result = @{
        Tenant     = $TenantName
        Version    = (Get-Date -Format 'yyyy-MM-ddTHH:mm:ss')
        SharePoint = @{
            Note = "TenantId not available via CLI (use Get-PnPTenantInfo in PnP PowerShell)"
        }
        Structure  = $script:Structure
    }
    
    Write-Host "`n==============================" -ForegroundColor Cyan
    Write-Host "Site Structure Analysis Complete" -ForegroundColor Cyan
    Write-Host "==============================" -ForegroundColor Cyan
    Write-Host "Hubs Processed: " -NoNewline
    Write-Host $script:Summary.HubsProcessed -ForegroundColor Green
    Write-Host "Sites Processed: " -NoNewline
    Write-Host $script:Summary.SitesProcessed -ForegroundColor Green
    Write-Host "Failures: " -NoNewline
    Write-Host $script:Summary.Failures -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "==============================`n" -ForegroundColor Cyan
    
    Stop-Transcript | Out-Null
    
    if ($AsObject) {
        return $result
    }
    else {
        Write-Host "`nSite Structure:" -ForegroundColor Cyan
        $result.Structure | Format-List
    }
}

```

# [PnP PowerShell](#tab/pnpps)

```powershell
# Initial parameters to start (connect to SPO admin center)
$adminUrl = "your SPO tenant admin url"
Connect-PnPOnline -Url $adminUrl -Interactive


Function Get-SiteStructure {
  param (
    [Parameter(HelpMessage = "The root site url to start with", Mandatory = $true)]
    [string] $RootSiteUrl,
    [Parameter(HelpMessage = "defines whether the site content (libraries) should be included", Mandatory = $false)]
    [switch] $WithSiteContent,
    [Parameter(HelpMessage = "defines whether the result will be returned as an object to be processed later", Mandatory = $false)]
    [switch] $AsObject
  )

  Function Initialize-Routine {
    $Script:RootSiteUrl = $RootSiteUrl
    $Script:Structure = @()
    $Script:TenantInfo = Get-PnPTenantInfo
    $Script:RetrieveSiteContent = $WithSiteContent
    
    Clear-Host
  }

  Function Get-SiteInfo([string] $SiteUrl) {
    $siteInfo = Get-PnPTenantSite -Identity $SiteUrl
    return @{
      Id        = $siteInfo.IsHubSite ? $siteInfo.HubSiteId : $siteInfo.Id
      Title     = $siteInfo.Title
      Url       = $siteInfo.Url
      Type      = $siteInfo.Template -eq "SITEPAGEPUBLISHING#0" ? "Communication" 
      : $siteInfo.Template -eq "GROUP#0" ? "Team" 
      : $siteInfo.Template -eq "STS#3" ? "SPOTeam" 
      : "Other"
      IsHubSite = $siteInfo.IsHubSite
    }
  }
  
  Function Get-StartSite {
    Function Get-HomeSite() {
      try {
        Write-Host "Trying to get the home site in your tenant and check whether it matches the root site"
        $homeSite = Get-PnPHomeSite -Detailed
        if (!$homeSite) { throw "Home Site not found or not set" }
        if ($homeSite.Url -ne $RootSiteUrl) { throw "Home Site does not match root site" }
        $siteInfo = Get-SiteInfo -SiteUrl $hubSite.SiteUrl
        $siteObject = ([Ordered]@{Hub = $siteInfo.Title; Url = $siteInfo.Url; Type = $siteInfo.Type })
        
        if ($Script:RetrieveSiteContent) { 
          $content = Get-SiteContent -RootSite $siteInfo 
          if ($content) { $siteObject.Content = $content }
        }

        $Script:Structure += $siteObject
        return $siteInfo
      }
      catch {
        throw $_
      }
    }
    
    Function Get-HubSite([string] $SiteUrl) {
      try {
        Write-Host "Trying to get the according hub site: " -NoNewline
        $hubSite = Get-PnPHubSite -Identity $RootSiteUrl
        if (!$hubSite) { throw "Hub Site not found or not set" }
        
        Write-Host -ForegroundColor DarkGreen "✔︎ Starting with $($hubSite.SiteUrl)"
        $siteInfo = Get-SiteInfo -SiteUrl $hubSite.SiteUrl
        $siteObject = ([Ordered]@{Hub = $siteInfo.Title; Url = $siteInfo.Url; Type = $siteInfo.Type })
        
        if ($Script:RetrieveSiteContent) { 
          $content = Get-SiteContent -RootSite $siteInfo 
          if ($content) { $siteObject.Content = $content }
        }

        $Script:Structure += $siteObject
        return $siteInfo
      }
      catch {
        throw $_
      }
    }
    
    # Run the start site routine
    try {
      return Get-HomeSite
    }
    catch {
      Write-Host -ForegroundColor DarkYellow $_
      return Get-HubSite -SiteUrl $RootSiteUrl
    }
  }

  Function Get-AssignedSites {
    param (
      [Parameter(HelpMessage = "the site from where the assigned sites will be retrieved", Mandatory = $true)]
      [object] $RootSite
    )
    
    # Get all hubs that are assigned to this site site;
    # either directly connected sites (first test) or connected hubs (second test)
    $children = (Get-PnPHubSiteChild -Identity $RootSite.Url) ?? (Get-PnPHubSite | ? { $_.ParentHubSiteId -eq $RootSite.Id })
    foreach ($site in $children) {
      $siteInfo = switch ($site.GetType().FullName) {
        "Microsoft.Online.SharePoint.TenantAdministration.SiteProperties" { Get-SiteInfo -SiteUrl $site.SiteUrl }
        "System.String" { Get-SiteInfo -SiteUrl $site }
        "Default" { Get-SiteInfo -SiteUrl $site }
      }

      Write-Host "👉 $($siteInfo.Url)"
      
      # Get all assigned sites in case of current site is a hub site
      if ($siteInfo.IsHubSite) {
        $siteObject = ([Ordered]@{Hub = $siteInfo.Title; Url = $siteInfo.Url; Type = $siteInfo.Type; ConnectedHubsite = $RootSite.Url })
        if ($Script:RetrieveSiteContent) { 
          $content = Get-SiteContent -RootSite $siteInfo 
          if ($content) { $siteObject.Content = $content }
        }
        
        $Script:Structure += $siteObject
        Get-AssignedSites -RootSite $siteInfo
      }
      else {
        $siteObject = ([Ordered]@{Site = $siteInfo.Title; Url = $siteInfo.Url; Type = $siteInfo.Type; ConnectedHubsite = $RootSite.Url })
        if ($Script:RetrieveSiteContent) { 
          $content = Get-SiteContent -RootSite $siteInfo 
          if ($content) { $siteObject.Content = $content }
        }
        
        $Script:Structure += $siteObject
      }
    }
  }

  Function Get-SiteContent {
    param (
      [Parameter(HelpMessage = "the site from where the content will be retrieved", Mandatory = $true)]
      [object] $RootSite
    )
    $output = @()
    $connSite = Connect-PnPOnline -Url $RootSite.Url -Interactive -ReturnConnection
    
    # Get the document libraries & lists
    $objects = Get-PnPList -Connection $connSite | `
      Where-Object { $_.BaseType -in @("DocumentLibrary", "GenericList", "Events") -and $_.Hidden -eq $false -and $_.EntityTypeName -notin @("Style_x0020_Library", "FormServerTemplates", "SiteAssets", "SitePages") }
    if ($objects) {
      foreach ($object in $objects) {
        $output += switch ($object.BaseType) {
          "DocumentLibrary" { [Ordered]@{ DocumentLibrary = $object.Title; Url = "/" + $object.RootFolder.ServerRelativeUrl.Split("/")[-1]; } }
          "GenericList" { [Ordered]@{ List = $object.Title; Url = "/" + $object.RootFolder.ServerRelativeUrl.Split("/")[-1]; } }
          "Default" { [Ordered]@{ List = $object.Title; Url = "/" + $object.RootFolder.ServerRelativeUrl.Split("/")[-1]; } }
        }
      }
    }
    $connSite = $null;
    return $output
  }
  
  #######################################
  # START Main Routine
  try {
    Initialize-Routine
    $startSite = Get-StartSite
    Get-AssignedSites -RootSite $startSite
    
    $result = [Ordered]@{
      Tenant     = $Script:TenantInfo.DisplayName
      Version    = (Get-Date).ToString("yyyy-MM-dd HH:mm:ss")
      SharePoint = @{TenantId = $Script:TenantInfo.TenantId.ToString() }
      Structure  = $Script:Structure
    }

    if ($AsObject.IsPresent) {
      return $result
    }
    else {
      $result.Structure
    }
  }
  catch {
    Write-Error $_
  }
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Source Credit

Sample taken from [https://github.com/tmaestrini/easyProvisioning/](https://github.com/tmaestrini/easyProvisioning)

## Contributors

| Author(s) |
|-----------|
| [Tobias Maestrini](https://github.com/tmaestrini)|
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-copy-hubsite-navigation" aria-hidden="true" />

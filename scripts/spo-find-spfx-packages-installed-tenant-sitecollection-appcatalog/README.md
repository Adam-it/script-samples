

# Find SharePoint Framework (SPFx) Packages in Tenant and Site Collection App Catalogs

## Summary

Find SPFx package installations within SharePoint Online environment in Tenant and Site Collection App Catalog using PnP PowerShell or CLI for Microsoft 365 which will help maintain oversight of SPFx package installation, ensuring they are up-to-date and compliant.

The script was particularly useful in pinpointing sites within the tenant where third-party applications, specifically an analytics SPFx component, were deployed. This was crucial for ensuring that data collection was confined to designated sites, such as the intranet in this case. Despite the analytics dashboard aggregating data from all tenant sites, it was challenging to discern the sources of data collection. Therefore, this script was developed to clearly identify the sites from which data were being collected.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have access to the SharePoint Online site.

# [PnP PowerShell](#tab/pnpps)

```powershell
# Parameters
$AdminCenterURL = "https://contosoonline-admin.sharepoint.com"
$tenantAppCatalogUrl = "https://contosoonline.sharepoint.com/sites/appcatalog"
$sppkgFolder = "./packages"
$dateTime = (Get-Date).toString("dd-MM-yyyy")
$fileName = "\InventorySPFx-" + $dateTime + ".csv"

$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$OutPutView = $directorypath + $fileName

 
cd $PSScriptRoot
$packageFiles = Get-ChildItem $sppkgFolder
 
Connect-PnPOnline $tenantAppCatalogUrl -Interactive
$appCatConnection  = Get-PnPConnection
 
Connect-PnPOnline $AdminCenterURL -Interactive
$adminConnection  = Get-PnPConnection
 
$SiteAppUpdateCollection = @()
 
#Get associated sites with hub
$associatedSites = Get-PnPTenantSite -Detailed -Connection $adminConnection  | Where-Object -Property Template -NotIn ("PWA#0","SRCHCEN#0", "REDIRECTSITE#0", "SPSMSITEHOST#0", "APPCATALOG#0", "POINTPUBLISHINGHUB#0", "POINTPUBLISHINGTOPIC#0","EDISC#0", "STS#-1") 
$istenantApp = ""

$associatedSites | select url | ForEach-Object {
  $Site = Get-PnPTenantSite $_.url -Connection $adminConnection
  Connect-PnPOnline -Url $Site.url -Interactive
  $siteConnection  = Get-PnPConnection   

  try{
  foreach($package in $packageFiles)
  {  
     $packageName = $package.PSChildName
     $appTitle = $null;
      #Find Name of app from installed package
      $RestMethodUrl = '/_api/web/lists/getbytitle(''Apps%20for%20SharePoint'')/items?$select=Title,LinkFilename'
      if((Get-PnPSiteCollectionAppCatalog -CurrentSite)){
        $apps = (Invoke-PnPSPRestMethod -Url $RestMethodUrl -Method Get -Connection $siteConnection).Value
        $appTitle = ($apps | where-object {$_.LinkFilename -eq $packageName} | select Title).Title
        $istenantApp = "Site Collection App Catalog"
      }
      if(!$appTitle)
      {
        $apps = (Invoke-PnPSPRestMethod -Url $RestMethodUrl -Method Get -Connection $appCatConnection).Value
        $appTitle = ($apps | where-object {$_.LinkFilename -eq $packageName} | select Title).Title
        $istenantApp = "Tenant App Catalog"
      }
  
    $web = Get-PnPWeb -Includes AppTiles -Connection $siteConnection
    $app = $web.AppTiles  |  where-object {$_.Title -eq $appTitle }
    $currentPackage = Get-PnPApp -Identity  $appTitle -Connection $siteConnection
    if($currentPackage.InstalledVersion){
      Write-Host "Current package version on site $($site.Url): $($currentPackage.InstalledVersion)"
      $ExportVw = New-Object PSObject
      $ExportVw | Add-Member -MemberType NoteProperty -name "Site URL" -value $Site.url
      $ExportVw | Add-Member -MemberType NoteProperty -name "Package Name" -value $packageName
        $ExportVw | Add-Member -MemberType NoteProperty -name "Hub Site Name" -value  (get-pnphubsite -identity $Site.HubSiteId.Guid).title
        $ExportVw | Add-Member -MemberType NoteProperty -name "Is Hub Site" -value $Site.IsHubSite
        $ExportVw | Add-Member -MemberType NoteProperty -name "Version" -value $currentPackage.InstalledVersion
        $ExportVw | Add-Member -MemberType NoteProperty -name "Is Tenant" -value $istenantApp
      $SiteAppUpdateCollection += $ExportVw
    }
  }
}
catch{
  write-host -f Red $_.Exception.Message
 }
}

#Export the result Array to CSV file
$SiteAppUpdateCollection | Export-CSV $OutPutView -Force -NoTypeInformation
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(HelpMessage="Filter sites to process (e.g., 'Url -like ''project''').")]
    [string]$SiteFilter,
    
    [Parameter(HelpMessage="Output directory for CSV report (default: current directory).")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $script:ReportCollection = @()
    $script:Summary = @{
        SitesProcessed = 0
        AppsFound = 0
        Failures = 0
    }
    $script:HubSiteCache = @{}
    
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    Start-Transcript -Path "SPFxInventory-$timestamp.log"
    
    Write-Verbose "Validating login status..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365"
    }
    
    Write-Host "Retrieving tenant app catalog URL..." -ForegroundColor Cyan
    $tenantAppCatalogUrl = m365 spo tenant appcatalogurl get --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve tenant app catalog URL. Ensure you have SharePoint Administrator permissions."
    }
    $tenantAppCatalogUrl = $tenantAppCatalogUrl | ConvertFrom-Json
    Write-Verbose "Tenant app catalog: $tenantAppCatalogUrl"
    
    Write-Host "Retrieving site collection app catalogs..." -ForegroundColor Cyan
    $siteAppCatalogsJson = m365 spo site appcatalog list --output json
    if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve site collection app catalogs. Continuing with tenant catalog only."
        $siteAppCatalogs = @()
    } else {
        $siteAppCatalogs = @($siteAppCatalogsJson | ConvertFrom-Json)
        Write-Verbose "Found $($siteAppCatalogs.Count) site collection app catalog(s)"
    }
    
    $script:SiteAppCatalogUrls = @{}
    foreach ($siteCat in $siteAppCatalogs) {
        $script:SiteAppCatalogUrls[$siteCat.AbsoluteUrl] = $true
    }
    
    Write-Host "Retrieving tenant app catalog apps..." -ForegroundColor Cyan
    $tenantAppsJson = m365 spo app list --appCatalogScope tenant --output json
    if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve tenant app catalog apps."
        $script:TenantApps = @()
    } else {
        $script:TenantApps = @($tenantAppsJson | ConvertFrom-Json)
        Write-Verbose "Found $($script:TenantApps.Count) app(s) in tenant catalog"
    }
    
    $script:SiteCollectionApps = @{}
    foreach ($siteCatUrl in $siteAppCatalogs.AbsoluteUrl) {
        Write-Verbose "Retrieving apps from site collection catalog: $siteCatUrl"
        $siteAppsJson = m365 spo app list --appCatalogScope sitecollection --appCatalogUrl $siteCatUrl --output json 2>&1
        if ($LASTEXITCODE -eq 0) {
            $siteApps = @($siteAppsJson | ConvertFrom-Json)
            $script:SiteCollectionApps[$siteCatUrl] = $siteApps
            Write-Verbose "Found $($siteApps.Count) app(s) in site collection catalog at $siteCatUrl"
        } else {
            Write-Warning "Failed to retrieve apps from $siteCatUrl"
        }
    }
}

process {
    Write-Host "Retrieving sites..." -ForegroundColor Cyan
    
    $sitesArgs = @('spo', 'site', 'list', '--output', 'json')
    if ($SiteFilter) {
        $sitesArgs += '--filter'
        $sitesArgs += $SiteFilter
    }
    
    $sitesJson = m365 @sitesArgs
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve sites. Ensure you have permissions to access the tenant admin site."
    }
    
    $sites = @($sitesJson | ConvertFrom-Json)
    
    $excludedTemplates = @('PWA#0', 'SRCHCEN#0', 'REDIRECTSITE#0', 'SPSMSITEHOST#0', 'APPCATALOG#0', 'POINTPUBLISHINGHUB#0', 'POINTPUBLISHINGTOPIC#0', 'EDISC#0', 'STS#-1')
    $sites = $sites | Where-Object { $_.Template -notin $excludedTemplates }
    
    Write-Host "Processing $($sites.Count) sites..." -ForegroundColor Cyan
    
    foreach ($site in $sites) {
        Write-Verbose "Processing site: $($site.Url)"
        
        try {
            $installedAppsJson = m365 spo app instance list --siteUrl $site.Url --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve installed apps for $($site.Url)"
                $script:Summary.Failures++
                continue
            }
            
            $installedApps = @($installedAppsJson | ConvertFrom-Json)
            
            if ($installedApps.Count -eq 0) {
                Write-Verbose "  No apps installed on $($site.Url)"
                $script:Summary.SitesProcessed++
                continue
            }
            
            Write-Host "  Found $($installedApps.Count) installed app(s) on $($site.Url)" -ForegroundColor Yellow
            
            foreach ($installedApp in $installedApps) {
                $catalogType = "Unknown"
                $packageName = $installedApp.Title
                $appCatalogVersion = ""
                
                $matchingTenantApp = $script:TenantApps | Where-Object { $_.ID -eq $installedApp.ProductId }
                if ($matchingTenantApp) {
                    $catalogType = "Tenant"
                    $packageName = $matchingTenantApp.Title
                    $appCatalogVersion = $matchingTenantApp.AppCatalogVersion
                } else {
                    foreach ($siteCatUrl in $script:SiteCollectionApps.Keys) {
                        $siteApps = $script:SiteCollectionApps[$siteCatUrl]
                        $matchingSiteApp = $siteApps | Where-Object { $_.ID -eq $installedApp.ProductId }
                        if ($matchingSiteApp) {
                            $catalogType = "Site Collection ($siteCatUrl)"
                            $packageName = $matchingSiteApp.Title
                            $appCatalogVersion = $matchingSiteApp.AppCatalogVersion
                            break
                        }
                    }
                }
                
                $hubSiteName = ""
                if ($site.HubSiteId -and $site.HubSiteId -ne "00000000-0000-0000-0000-000000000000") {
                    if (-not $script:HubSiteCache.ContainsKey($site.HubSiteId)) {
                        try {
                            $hubSiteJson = m365 spo hubsite get --id $site.HubSiteId --output json 2>&1
                            if ($LASTEXITCODE -eq 0) {
                                $hubSite = $hubSiteJson | ConvertFrom-Json
                                $script:HubSiteCache[$site.HubSiteId] = $hubSite.Title
                            } else {
                                $script:HubSiteCache[$site.HubSiteId] = ""
                            }
                        }
                        catch {
                            $script:HubSiteCache[$site.HubSiteId] = ""
                        }
                    }
                    $hubSiteName = $script:HubSiteCache[$site.HubSiteId]
                }
                
                $exportItem = [PSCustomObject]@{
                    'Site URL' = $site.Url
                    'Site Title' = $site.Title
                    'Package Name' = $packageName
                    'Hub Site Name' = $hubSiteName
                    'Is Hub Site' = $site.IsHubSite
                    'Installed Version' = $installedApp.Version
                    'Catalog Version' = $appCatalogVersion
                    'Catalog Type' = $catalogType
                    'Product ID' = $installedApp.ProductId
                    'App ID' = $installedApp.AppId
                }
                
                $script:ReportCollection += $exportItem
                $script:Summary.AppsFound++
                Write-Verbose "    - $packageName (v$($installedApp.Version)) from $catalogType"
            }
            
            $script:Summary.SitesProcessed++
        }
        catch {
            Write-Warning "Error processing $($site.Url): $_"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Host "`nSummary:" -ForegroundColor Cyan
    Write-Host "  Sites processed: $($script:Summary.SitesProcessed)" -ForegroundColor Green
    Write-Host "  Apps found: $($script:Summary.AppsFound)" -ForegroundColor Green
    if ($script:Summary.Failures -gt 0) {
        Write-Host "  Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
    
    if ($script:ReportCollection.Count -gt 0) {
        $csvTimestamp = Get-Date -Format "yyyyMMdd-HHmmss"
        $csvPath = Join-Path $OutputPath "SPFxInventory-$csvTimestamp.csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Force
        Write-Host "`nReport exported to: $csvPath" -ForegroundColor Green
    } else {
        Write-Host "`nNo SPFx packages found installed in any site." -ForegroundColor Yellow
    }
    
    Stop-Transcript
}

# Example 1: Find all SPFx packages across all sites
# .\Find-SPFxPackages.ps1

# Example 2: Filter specific sites
# .\Find-SPFxPackages.ps1 -SiteFilter "Url -like 'project'" -Verbose

# Example 3: Custom output directory
# .\Find-SPFxPackages.ps1 -OutputPath "C:\Reports"

# Example 4: Combine filter with custom output
# .\Find-SPFxPackages.ps1 -SiteFilter "Template eq 'GROUP#0'" -OutputPath "C:\Reports" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

Sample first appeared on [Find SharePoint Framework (SPFx) Packages with PowerShell in Tenant and Site Collection App Catalogs](https://reshmeeauckloo.com/posts/powershell_find-spfx-installs-in-tenant-sitecollection-appcatalog/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-find-spfx-packages-installed-tenant-sitecollection-appcatalog" aria-hidden="true" />

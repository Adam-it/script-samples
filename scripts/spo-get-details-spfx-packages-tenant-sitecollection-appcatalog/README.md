

# Retrieve SPFx Details from Tenant and Site Collection App Catalogs

## Summary

This script will help to gather detailed information about SPFx solutions installed in SharePoint environment, such as API permissions, for auditing, inventory, or compliance purposes from both the tenant-level and site collection app catalogs.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have access to the SharePoint Online site.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint admin center URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$AdminCenterUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Output directory for CSV report")]
    [ValidateScript({Test-Path $_ -PathType Container})]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "dd-MM-yyyy-HHmmss"
    $transcriptPath = Join-Path $OutputPath "SPFxInventory-Transcript-$timestamp.log"
    Start-Transcript -Path $transcriptPath -NoClobber

    Write-Host "Starting SPFx Package Inventory..." -ForegroundColor Cyan
    Write-Verbose "Output path: $OutputPath"
    
    Write-Host "Ensuring CLI for Microsoft 365 authentication..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    Write-Host "Authentication successful" -ForegroundColor Green
    
    $script:AppDetails = @()
    $script:Summary = @{
        TenantApps = 0
        SiteCollections = 0
        SiteCollectionApps = 0
        Failures = 0
    }
}

process {
    Write-Host "Retrieving tenant app catalog URL..." -ForegroundColor Cyan
    $tenantCatalogUrl = m365 spo tenant appcatalogurl get --output text 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve tenant app catalog URL. Error: $tenantCatalogUrl"
    }
    Write-Verbose "Tenant app catalog URL: $tenantCatalogUrl"
    
    Write-Host "Retrieving apps from tenant app catalog..." -ForegroundColor Cyan
    $tenantAppsJson = m365 spo app list --output json 2>&1
    if ($LASTEXITCODE -eq 0) {
        $tenantApps = @($tenantAppsJson | ConvertFrom-Json)
        Write-Host "Found $($tenantApps.Count) app(s) in tenant catalog" -ForegroundColor Green
        
        foreach ($app in $tenantApps) {
            $app | Add-Member -MemberType NoteProperty -Name 'SiteUrl' -Value $tenantCatalogUrl
            $app | Add-Member -MemberType NoteProperty -Name 'CatalogScope' -Value 'Tenant'
            $script:AppDetails += $app
        }
        $script:Summary.TenantApps = $tenantApps.Count
    } else {
        Write-Warning "Failed to retrieve apps from tenant catalog: $tenantAppsJson"
        $script:Summary.Failures++
    }
    
    Write-Host "Retrieving site collection app catalogs..." -ForegroundColor Cyan
    $siteCollectionsJson = m365 spo site appcatalog list --excludeDeletedSites --output json 2>&1
    if ($LASTEXITCODE -eq 0) {
        $siteCollections = @($siteCollectionsJson | ConvertFrom-Json)
        Write-Host "Found $($siteCollections.Count) site collection app catalog(s)" -ForegroundColor Green
        $script:Summary.SiteCollections = $siteCollections.Count
        
        $counter = 0
        foreach ($site in $siteCollections) {
            $counter++
            Write-Host "[$counter/$($siteCollections.Count)] Processing: $($site.AbsoluteUrl)" -ForegroundColor Yellow
            
            try {
                $siteAppsJson = m365 spo app list --appCatalogScope sitecollection --appCatalogUrl $site.AbsoluteUrl --output json 2>&1
                if ($LASTEXITCODE -eq 0) {
                    $siteApps = @($siteAppsJson | ConvertFrom-Json)
                    Write-Verbose "Found $($siteApps.Count) app(s) in site collection"
                    
                    foreach ($app in $siteApps) {
                        $app | Add-Member -MemberType NoteProperty -Name 'SiteUrl' -Value $site.AbsoluteUrl
                        $app | Add-Member -MemberType NoteProperty -Name 'CatalogScope' -Value 'SiteCollection'
                        $script:AppDetails += $app
                    }
                    $script:Summary.SiteCollectionApps += $siteApps.Count
                } else {
                    Write-Warning "Failed to list apps from $($site.AbsoluteUrl): $siteAppsJson"
                    $script:Summary.Failures++
                }
            } catch {
                Write-Warning "Error processing site $($site.AbsoluteUrl): $_"
                $script:Summary.Failures++
            }
        }
    } else {
        Write-Warning "Failed to retrieve site collection app catalogs: $siteCollectionsJson"
        $script:Summary.Failures++
    }
}

end {
    $csvPath = Join-Path $OutputPath "InventorySPFx-$timestamp.csv"
    
    if ($script:AppDetails.Count -gt 0) {
        Write-Host "Exporting results to CSV..." -ForegroundColor Cyan
        $script:AppDetails | Export-Csv -Path $csvPath -NoTypeInformation -Force
        Write-Host "CSV exported: $csvPath" -ForegroundColor Green
    } else {
        Write-Warning "No apps found. CSV will not be created."
    }
    
    Write-Host "`n================================" -ForegroundColor Cyan
    Write-Host "SPFx Package Inventory Summary" -ForegroundColor Cyan
    Write-Host "================================" -ForegroundColor Cyan
    Write-Host "Tenant Apps: $($script:Summary.TenantApps)" -ForegroundColor White
    Write-Host "Site Collections: $($script:Summary.SiteCollections)" -ForegroundColor White
    Write-Host "Site Collection Apps: $($script:Summary.SiteCollectionApps)" -ForegroundColor White
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Green
    }
    
    Write-Host "Total Apps: $($script:AppDetails.Count)" -ForegroundColor Cyan
    if ($script:AppDetails.Count -gt 0) {
        Write-Host "Report: $csvPath" -ForegroundColor Cyan
    }
    Write-Host "Transcript: $transcriptPath" -ForegroundColor Cyan
    Write-Host "================================`n" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Basic usage - Retrieve SPFx packages from all catalogs
# .\Get-SPFxPackageInventory.ps1 -AdminCenterUrl "https://contoso-admin.sharepoint.com"

# Specify custom output directory
# .\Get-SPFxPackageInventory.ps1 -AdminCenterUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\Reports"

# Run with verbose output to see detailed progress
# .\Get-SPFxPackageInventory.ps1 -AdminCenterUrl "https://contoso-admin.sharepoint.com" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell
$AdminCenterURL= Read-Host -Prompt "Enter admin tenant collection URL";

$tenantAppCatalogUrl = Get-PnPTenantAppCatalogUrl
$dateTime = (Get-Date).toString("dd-MM-yyyy")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "\InventorySPFx-" + $dateTime + ".csv"
$OutPutView = $directorypath + $fileName
 
cd $PSScriptRoot
 
Connect-PnPOnline $tenantAppCatalogUrl -Interactive
$appCatConnection  = Get-PnPConnection
 
Connect-PnPOnline $AdminCenterURL -Interactive
$adminConnection  = Get-PnPConnection
 
$appsDetails = @()
 
#Get associated sites with hub
$sites = Get-PnPTenantSite -Detailed -Connection $adminConnection  | Where-Object -Property Template -NotIn ("PWA#0","SRCHCEN#0", "REDIRECTSITE#0", "SPSMSITEHOST#0", "APPCATALOG#0", "POINTPUBLISHINGHUB#0", "POINTPUBLISHINGTOPIC#0","EDISC#0", "STS#-1") 
$RestMethodUrl = '/_api/web/lists/getbytitle(''Apps%20for%20SharePoint'')/items?$select=Title,LinkFilename,SkipFeatureDeployment,ContainsTeamsManifest,ContainsVivaManifest,SupportsTeamsTabs,WebApiPermissionScopesNote,ContainsTenantWideExtension,IsolatedDomain,PackageDefaultSkipFeatureDeployment,IsClientSideSolutionCurrentVersionDeployed,ExternalContentDomains,IsClientSideSolutionDeployed,IsClientSideSolution,AppPackageErrorMessage,IsValidAppPackage,SharePointAppCategory,AppDescription,AppShortDescription'

$apps = (Invoke-PnPSPRestMethod -Url $RestMethodUrl -Method Get -Connection $appCatConnection).Value
#export details of apps
$apps| foreach-object{
    $app = $_
    $app  | Add-Member -MemberType NoteProperty -name "Site Url" -value $tenantAppCatalogUrl
    $appsDetails += $app
}

$sites | select url | ForEach-Object {
  write-host "Processing Site:" $_.url -f Yellow
    $Site = Get-PnPTenantSite $_.url -Connection $adminConnection
  Connect-PnPOnline -Url $Site.url -Interactive
  $siteConnection  = Get-PnPConnection   

  try{
      if((Get-PnPSiteCollectionAppCatalog -CurrentSite)){
        $apps = (Invoke-PnPSPRestMethod -Url $RestMethodUrl -Method Get -Connection $siteConnection).Value
        $apps| foreach-object{
            $app = $_
            $app  | Add-Member -MemberType NoteProperty -name "Site Url" -value $Site.url
            $appsDetails += $app
        }
      }
     }
 catch{
  write-host -f Red $_.Exception.Message
 }
}
#Export the result Array to CSV file
$appsDetails | Export-CSV $OutPutView -Force -NoTypeInformation
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Retrieve SPFx Details from Tenant and Site Collection App Catalogs Using PowerShell](https://reshmeeauckloo.com/posts/powershell-get-spfx-details-tenant-sitecollection-appcatalog/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-details-spfx-packages-tenant-sitecollection-appcatalog" aria-hidden="true" />

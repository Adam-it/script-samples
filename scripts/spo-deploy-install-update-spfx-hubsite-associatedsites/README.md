

# Deploys and Installs SharePoint Framework (SPFx) solutions to Hub Site and Associated Sites using Site Collection App Catalog

## Summary

At the time of submitting this script sample there is no concept of a hub site app catalog. However you may want to install or upgrade a SPFx solution to all sites within a hub for example all sites linked to the intranet hub. This sample is applicable for a SPFx solution that needs to be deployed and upgraded across all sites in a hub using Site Collection App Catalog. Available in both PnP PowerShell and CLI for Microsoft 365 v11.4.0+.

![Example Screenshot](assets/example.png)

### Prerequisites

- The user account that runs the script must have SharePoint Online tenant administrator access.
- Before running the script, edit the script and update the variable values in the Config Variables section, such as Admin Center URL, Hub Site URL, the CSV output file path and alternatively the sppkg packages Folder. 

The script will:
- Get the hub site ID using the Get-PnPHubSite cmdlet.
- Get all associated site collections in the tenant using the Get-PnPTenantSite cmdlet filtered by the HubSiteId property.
- Connect to the site using Connect-PnPOnline.
- Check if the site collection app catalog exists. If it doesn’t, create it using Add-PnPSiteCollectionAppCatalog.
- Deploy the SPFx package using Add-PnPApp.
- Check if the package is already installed on the site using Get-PnPApp.
- If the package is not installed, install it using Install-PnPApp.
- If a newer version of the package is available, update the package using Update-PnPApp.
- Export the site collection URL, package name, and package version to a CSV file for a record of what's updated.

The script does not cover admin consent to app permissions. A global administrator will have to grant admin consent if required by the SPFx solutions. 

# [PnP PowerShell](#tab/pnpps)

```powershell
#Config variables
$adminCenterURL = "https://tenant-admin.sharepoint.com"
$fileName = "\IntranetUpgradeSPFx-" + $dateTime + ".csv"
$hubSiteUrl = "https://tenant.sharepoint.com/sites/u-intranet"
$OutPutView = $directorypath + $fileName
$sppkgFolder = "./packages"

$dateTime = (Get-Date).toString("dd-MM-yyyy")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
cd $PSScriptRoot

$packageFiles = Get-ChildItem $sppkgFolder

Connect-PnPOnline $adminCenterURL -Interactive

#collection to save the list of sites where the deployment or upgrades of SPFx solution happened for auditing
$ViewCollection = @() 
$HubSite = Get-PnPHubSite -Identity $hubSiteUrl
$associatedSites = Get-PnPTenantSite -Detailed | Where-Object {$_.HubSiteId -eq $hubSite.Id}

#Get all site collections associated with the hub site
$associatedSites | select url | ForEach-Object { 
  $Site = Get-PnPTenantSite $_.url
  If($Site.HubSiteId -eq $HubSiteId){
    Connect-PnPOnline -Url $Site.url -Interactive
    Write-Host ("Deploying packages to {0}..." -f $Site.url) -ForegroundColor Yellow

      foreach($package in $packageFiles){
        $ExportVw = New-Object PSObject
        $ExportVw | Add-Member -MemberType NoteProperty -name "Site URL" -value $Site.url
        $packageName = $package.PSChildName
        Write-Host ("Installing {0}..." -f $packageName) -ForegroundColor Yellow
         $ExportVw | Add-Member -MemberType NoteProperty -name "Package Name" -value $packageName
         #ensure app catalog
          if(!(Get-PnPSiteCollectionAppCatalog -CurrentSite)){
            Write-Host ("Creating site collection app catalog in {0}..." -f $Site.url) -ForegroundColor Yellow
            Add-PnPSiteCollectionAppCatalog
          }
         while(!(Get-PnPSiteCollectionAppCatalog -CurrentSite)){
            Start-Sleep -Seconds 20
         }
         #deploy sppkg
         Add-PnPApp -Path ("{0}/{1}" -f $sppkgFolder , $package.PSChildName) -Scope Site -Overwrite -Publish

         Start-Sleep -Seconds 5

        #Find Name of app from installed package 
        $RestMethodUrl = '/_api/web/lists/getbytitle(''Apps%20for%20SharePoint'')/items?$select=Title,LinkFilename'
        $apps = (Invoke-PnPSPRestMethod -Url $RestMethodUrl -Method Get).Value
        $appTitle = ($apps | where-object {$_.LinkFilename -eq $packageName} | select Title).Title
        # Get the current version of the SPFx package
        $currentPackage = Get-PnPApp -Identity $appTitle -Scope Site
        Write-Host "Current package version on site $($site.Url): $($currentPackage.InstalledVersion)"

        #Install App to the Site if not already installed
        $web = Get-PnPWeb -Includes AppTiles
        $app = $web.AppTiles  |  where-object {$_.Title -eq $currentPackage.Title } 
        if(!$app){
            Install-PnPApp -Identity $currentPackage.Id -Scope Site
            Start-Sleep -Seconds 5
        }
        # Get the latest version of the SPFx package
        Write-Host "Latest package version: $($currentPackage.AppCatalogVersion)"

        # Update the package to the latest version
        if ($currentPackage.InstalledVersion -ne $currentPackage.AppCatalogVersion) {
            Write-Host "Upgrading package on site $($site.Url) to latest version..." -ForegroundColor Green
            Update-PnPApp -Identity $currentPackage.Id -Scope site
            $currentPackage = Get-PnPApp -Identity $appTitle -Scope Site
            $ExportVw | Add-Member -MemberType NoteProperty -name "Package Version" -value $currentPackage.AppCatalogVersion
            $ViewCollection += $ExportVw
        } else {
            Write-Host "Package already up-to-date on site $($site.Url)."
        }
    }
  }
}

#Export the result Array to CSV file
$ViewCollection | Export-CSV $OutPutView -Force -NoTypeInformation
Disconnect-PnPOnline
```

> [!Note]
> SharePoint admin rights are required to run the script

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage="Hub site URL (e.g., 'https://contoso.sharepoint.com/sites/intranet')")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$HubSiteUrl,
    
    [Parameter(Mandatory, HelpMessage="Path to folder containing .sppkg files")]
    [ValidateScript({Test-Path $_ -PathType Container})]
    [string]$PackageFolder,
    
    [Parameter(HelpMessage="Output path for CSV report")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Host "Starting SPFx deployment to hub site and associated sites..." -ForegroundColor Cyan
    
    $transcriptPath = Join-Path $OutputPath "SPFxDeployment_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    Start-Transcript -Path $transcriptPath | Out-Null
    
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate to Microsoft 365"
    }
    Write-Verbose "Successfully authenticated to Microsoft 365"
    
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath -PathType Container)) {
            throw "Output path '$OutputPath' does not exist or is not a directory"
        }
    }
    
    $script:ReportCollection = [System.Collections.ArrayList]::new()
    $script:Summary = @{
        SitesProcessed = 0
        AppsDeployed = 0
        AppsInstalled = 0
        AppsUpgraded = 0
        Failures = 0
    }
    
    Write-Host "Scanning package folder..." -ForegroundColor Cyan
    $script:PackageFiles = Get-ChildItem -Path $PackageFolder -Filter "*.sppkg"
    if ($script:PackageFiles.Count -eq 0) {
        throw "No .sppkg files found in $PackageFolder"
    }
    Write-Host "Found $($script:PackageFiles.Count) package(s) to deploy" -ForegroundColor White
    
    Write-Host "Retrieving hub site and associated sites..." -ForegroundColor Cyan
    $hubJson = m365 spo hubsite get --url $HubSiteUrl --withAssociatedSites --output json
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve hub site information"
    }
    
    $hubData = $hubJson | ConvertFrom-Json
    $script:AssociatedSites = @($hubData.AssociatedSites)
    Write-Host "Found $($script:AssociatedSites.Count) associated site(s)" -ForegroundColor White
}

process {
    foreach ($site in $script:AssociatedSites) {
        $siteUrl = $site.SiteUrl
        $script:Summary.SitesProcessed++
        
        Write-Host "`nProcessing site: $siteUrl" -ForegroundColor Cyan
        
        try {
            Write-Verbose "Ensuring site collection app catalog exists..."
            $addCatalogResult = m365 spo site appcatalog add --siteUrl $siteUrl --output text 2>&1
            
            if ($LASTEXITCODE -eq 0) {
                Write-Host "  ✓ Site collection app catalog created" -ForegroundColor Green
            }
            elseif ($addCatalogResult -like "*already exists*" -or $addCatalogResult -like "*already enabled*") {
                Write-Verbose "  App catalog already exists"
                Write-Host "  ✓ Site collection app catalog ready" -ForegroundColor Green
            }
            else {
                Write-Warning "Failed to ensure app catalog for $siteUrl: $addCatalogResult"
                $script:Summary.Failures++
                continue
            }
            
            foreach ($package in $script:PackageFiles) {
                $packageName = $package.Name
                $packagePath = $package.FullName
                $packageBaseName = $package.BaseName
                
                Write-Host "  Processing package: $packageName" -ForegroundColor Yellow
                
                try {
                    Write-Verbose "    Uploading package to site collection app catalog..."
                    $null = m365 spo app add --filePath $packagePath --appCatalogScope sitecollection --appCatalogUrl $siteUrl --overwrite --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        throw "Failed to upload package"
                    }
                    $script:Summary.AppsDeployed++
                    
                    Write-Verbose "    Deploying (publishing) package..."
                    $null = m365 spo app deploy --name $packageName --appCatalogScope sitecollection --appCatalogUrl $siteUrl 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        throw "Failed to deploy package"
                    }
                    
                    Start-Sleep -Seconds 5
                    
                    Write-Verbose "    Retrieving app details from catalog..."
                    $appJson = m365 spo app list --appCatalogUrl $siteUrl --appCatalogScope sitecollection --output json
                    if ($LASTEXITCODE -ne 0) {
                        throw "Failed to retrieve app list"
                    }
                    
                    $apps = @($appJson | ConvertFrom-Json)
                    $app = $apps | Where-Object { $_.Title -like "*$packageBaseName*" } | Select-Object -First 1
                    
                    if (-not $app) {
                        Write-Warning "    Could not find app in catalog after deployment"
                        $script:Summary.Failures++
                        
                        $reportEntry = [PSCustomObject]@{
                            SiteUrl = $siteUrl
                            PackageName = $packageName
                            AppTitle = "Not Found"
                            InstalledVersion = "N/A"
                            CatalogVersion = "N/A"
                            Status = "Failed: App not found in catalog"
                        }
                        $null = $script:ReportCollection.Add($reportEntry)
                        continue
                    }
                    
                    Write-Verbose "    Checking if app is already installed on site..."
                    $instancesJson = m365 spo app instance list --siteUrl $siteUrl --output json
                    if ($LASTEXITCODE -ne 0) {
                        throw "Failed to retrieve app instances"
                    }
                    
                    $instances = @($instancesJson | ConvertFrom-Json)
                    $installedApp = $instances | Where-Object { $_.Title -eq $app.Title }
                    
                    $status = ""
                    if (-not $installedApp) {
                        Write-Host "    ✓ Installing app..." -ForegroundColor Green
                        $null = m365 spo app install --id $app.Id --siteUrl $siteUrl --appCatalogScope sitecollection 2>&1
                        if ($LASTEXITCODE -eq 0) {
                            $script:Summary.AppsInstalled++
                            $status = "Installed"
                            Write-Verbose "      Successfully installed app (v$($app.AppCatalogVersion))"
                        }
                        else {
                            throw "Failed to install app"
                        }
                    }
                    elseif ($installedApp.InstalledVersion -ne $app.AppCatalogVersion) {
                        Write-Host "    ✓ Upgrading from v$($installedApp.InstalledVersion) to v$($app.AppCatalogVersion)..." -ForegroundColor Green
                        $null = m365 spo app upgrade --id $app.Id --siteUrl $siteUrl --appCatalogScope sitecollection 2>&1
                        if ($LASTEXITCODE -eq 0) {
                            $script:Summary.AppsUpgraded++
                            $status = "Upgraded"
                            Write-Verbose "      Successfully upgraded app"
                        }
                        else {
                            throw "Failed to upgrade app"
                        }
                    }
                    else {
                        Write-Host "    ✓ Already up-to-date (v$($app.AppCatalogVersion))" -ForegroundColor Gray
                        $status = "Up-to-date"
                    }
                    
                    $reportEntry = [PSCustomObject]@{
                        SiteUrl = $siteUrl
                        PackageName = $packageName
                        AppTitle = $app.Title
                        InstalledVersion = if ($installedApp) { $installedApp.InstalledVersion } else { $app.AppCatalogVersion }
                        CatalogVersion = $app.AppCatalogVersion
                        Status = $status
                    }
                    $null = $script:ReportCollection.Add($reportEntry)
                }
                catch {
                    Write-Warning "    Failed to process $packageName on $siteUrl: $($_.Exception.Message)"
                    $script:Summary.Failures++
                    
                    $reportEntry = [PSCustomObject]@{
                        SiteUrl = $siteUrl
                        PackageName = $packageName
                        AppTitle = "N/A"
                        InstalledVersion = "N/A"
                        CatalogVersion = "N/A"
                        Status = "Failed: $($_.Exception.Message)"
                    }
                    $null = $script:ReportCollection.Add($reportEntry)
                    continue
                }
            }
        }
        catch {
            Write-Warning "Error processing site $siteUrl: $($_.Exception.Message)"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Host "`n=== Deployment Summary ===" -ForegroundColor Cyan
    Write-Host "Sites processed: $($script:Summary.SitesProcessed)" -ForegroundColor White
    Write-Host "Apps deployed: $($script:Summary.AppsDeployed)" -ForegroundColor Green
    Write-Host "Apps installed: $($script:Summary.AppsInstalled)" -ForegroundColor Green
    Write-Host "Apps upgraded: $($script:Summary.AppsUpgraded)" -ForegroundColor Green
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
    else {
        Write-Host "Failures: 0" -ForegroundColor Green
    }
    
    if ($script:ReportCollection.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "SPFxDeployment_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "`nReport exported to: $csvPath" -ForegroundColor Green
    }
    
    Stop-Transcript | Out-Null
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Green
}

# .\Deploy-SPFxToHub.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/intranet" -PackageFolder "./packages"

# .\Deploy-SPFxToHub.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/intranet" -PackageFolder "./packages" -Verbose

# .\Deploy-SPFxToHub.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/intranet" -PackageFolder "./packages" -OutputPath "C:\\Reports"

# .\Deploy-SPFxToHub.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/intranet" -PackageFolder "C:\\SPFx\\packages"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

Sample first appeared on [Deploying and Installing SharePoint Framework (SPFx) solutions using PnP PowerShell to Hub Site and Associated Sites](https://pnp.github.io/blog/post/deploy-spfx-in-hub-site-and-associated-sites/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-deploy-install-update-spfx-hubsite-associatedsites" aria-hidden="true" />

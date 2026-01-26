

# Deploy sppkgs and install apps

## Summary

Deploy all packages from a local folder and install the apps to the SharePoint site using PnP PowerShell or CLI for Microsoft 365.

![Example Screenshot](assets/example.png)

# [PnP PowerShell](#tab/pnpps)

```powershell

param (
  [Parameter(Mandatory = $true)]
  [string]$url,
  [Parameter(Mandatory = $true)]
  [string]$username,
  [Parameter(Mandatory = $true)]
  [string]$password
)

clear-host

# Connect
$psw = ConvertTo-SecureString -String $password -AsPlainText -Force
$credentials = New-Object -TypeName System.Management.Automation.PSCredential -argumentlist $UserName, $psw
Connect-PnPOnline -Url $url -Credentials $credentials

# Local sppkg folder path
$sppkgFolder = "./packages"

Write-Host ("Deploying packages to {0}..." -f $url) -ForegroundColor Yellow

$packagesFiles = Get-ChildItem $sppkgFolder

foreach ($package in $packagesFiles) {
  Write-Host ("Installing {0}..." -f $package.PSChildName) -ForegroundColor Yellow

  # Deploy sppkg
  $App = Add-PnPApp -Path ("{0}/{1}" -f $sppkgFolder, $package.PSChildName) -Scope Site -Publish -Overwrite

  #Install app
  if($null -eq $App.InstalledVersion) {
    Install-PnPApp -Identity $App.Id -Scope Site
  }
}

Disconnect-PnPOnline

Write-Host ("DONE") -ForegroundColor Green

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param (
  [Parameter(Mandatory, HelpMessage="The URL of the site collection with the app catalog")]
  [string]$SiteUrl,

  [Parameter(HelpMessage="Path to the folder containing .sppkg files")]
  [string]$PackageFolderPath = "./packages"
)

begin {
  $script:Summary = @{
    Deployed = 0
    Installed = 0
    Upgraded = 0
    Skipped = 0
    Failures = 0
  }

  Start-Transcript -Path "spo-deploy-apps-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"

  Write-Verbose "Ensuring CLI for Microsoft 365 login..."
  m365 login --ensure
  if ($LASTEXITCODE -ne 0) { throw "Failed to authenticate with CLI for Microsoft 365" }

  Write-Verbose "Validating package folder path..."
  if (-not (Test-Path $PackageFolderPath)) {
    throw "Package folder not found: $PackageFolderPath"
  }
}

process {
  Write-Host "\nDeploying packages to $SiteUrl..." -ForegroundColor Cyan

  $packages = Get-ChildItem $PackageFolderPath -Filter *.sppkg

  if ($packages.Count -eq 0) {
    Write-Warning "No .sppkg files found in $PackageFolderPath"
    return
  }

  Write-Host "Found $($packages.Count) package(s) to process" -ForegroundColor Green

  foreach ($package in $packages) {
    Write-Verbose "Processing $($package.Name)..."

    if ($PSCmdlet.ShouldProcess($package.Name, 'Deploy and install app')) {
      try {
        Write-Host "  Uploading $($package.Name)..." -ForegroundColor Yellow
        $addJson = m365 spo app add --filePath $package.FullName --appCatalogScope sitecollection --appCatalogUrl $SiteUrl --overwrite --output json
        if ($LASTEXITCODE -ne 0) { throw "Failed to upload app" }
        $app = $addJson | ConvertFrom-Json

        Write-Host "  Deploying $($package.Name)..." -ForegroundColor Yellow
        m365 spo app deploy --id $app.UniqueId --appCatalogScope sitecollection --appCatalogUrl $SiteUrl
        if ($LASTEXITCODE -ne 0) { throw "Failed to deploy app" }
        $script:Summary.Deployed++

        Write-Verbose "Checking if $($package.Name) is already installed..."
        $getJson = m365 spo app get --id $app.UniqueId --appCatalogScope sitecollection --appCatalogUrl $SiteUrl --output json
        if ($LASTEXITCODE -ne 0) { throw "Failed to get app info" }
        $appInfo = $getJson | ConvertFrom-Json

        if ([string]::IsNullOrEmpty($appInfo.InstalledVersion)) {
          Write-Host "  Installing $($package.Name)..." -ForegroundColor Yellow
          m365 spo app install --id $app.UniqueId --siteUrl $SiteUrl --appCatalogScope sitecollection
          if ($LASTEXITCODE -ne 0) { throw "Failed to install app" }
          $script:Summary.Installed++
        } elseif ($appInfo.InstalledVersion -ne $appInfo.AppCatalogVersion) {
          Write-Host "  Upgrading $($package.Name) from v$($appInfo.InstalledVersion) to v$($appInfo.AppCatalogVersion)..." -ForegroundColor Yellow
          m365 spo app upgrade --id $app.UniqueId --siteUrl $SiteUrl --appCatalogScope sitecollection
          if ($LASTEXITCODE -ne 0) { throw "Failed to upgrade app" }
          $script:Summary.Upgraded++
        } else {
          Write-Verbose "App already up-to-date: $($package.Name) v$($appInfo.InstalledVersion)"
          $script:Summary.Skipped++
        }

        Write-Host "  ✓ Completed: $($package.Name)" -ForegroundColor Green
      }
      catch {
        Write-Warning "Failed to process $($package.Name): $_"
        $script:Summary.Failures++
        continue
      }
    }
  }
}

end {
  Write-Host "\nSummary:" -ForegroundColor Cyan
  Write-Host "  Deployed: $($script:Summary.Deployed)" -ForegroundColor Green
  Write-Host "  Installed: $($script:Summary.Installed)" -ForegroundColor Green
  Write-Host "  Upgraded: $($script:Summary.Upgraded)" -ForegroundColor Green
  Write-Host "  Skipped (already installed): $($script:Summary.Skipped)" -ForegroundColor Yellow
  if ($script:Summary.Failures -gt 0) {
    Write-Host "  Failures: $($script:Summary.Failures)" -ForegroundColor Red
  }
  Stop-Transcript
}

# Example 1: Deploy and install all packages
# .\Deploy-SPFxApps.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/appcatalog"

# Example 2: Custom package folder with WhatIf
# .\Deploy-SPFxApps.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/appcatalog" -PackageFolderPath "C:\packages" -WhatIf

# Example 3: Verbose output
# .\Deploy-SPFxApps.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/appcatalog" -Verbose

# Example 4: Deploy and automatically upgrade existing apps to newer versions
# .\Deploy-SPFxApps.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/appcatalog"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

---

## Contributors

| Author(s)                                 |
| ----------------------------------------- |
| [Matteo Serpi](https://github.com/srpmtt) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-deploy-sppkgs-and-install-apps" aria-hidden="true" />

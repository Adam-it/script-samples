

# Download sppkgs from App Catalog

## Summary

Download all .sppkg packages from the SharePoint App Catalog for backup, migration, or version control purposes.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "App Catalog site URL (e.g., https://contoso.sharepoint.com/sites/apps)")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$AppCatalogUrl,

    [Parameter(HelpMessage = "Local folder path to save .sppkg files (default: .\\pkg)")]
    [string]$OutputPath = ".\\pkg"
)

begin {
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to Microsoft 365"
    }

    if (Test-Path -Path $OutputPath) {
        Get-ChildItem -Path $OutputPath -Filter *.sppkg -File | Remove-Item -Force
    } else {
        New-Item -Path $OutputPath -ItemType Directory | Out-Null
    }

    $script:Summary = @{
        FilesFound  = 0
        Downloaded  = 0
        Failures    = 0
    }

    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    Start-Transcript -Path "DownloadSPPKGs_$timestamp.log"
}

process {
    try {
        Write-Host "Listing .sppkg files in app catalog..." -ForegroundColor Cyan
        
        $filesJson = m365 spo file list --webUrl $AppCatalogUrl --folderUrl "AppCatalog" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list files in app catalog. Verify URL and permissions."
        }

        $allFiles = $filesJson | ConvertFrom-Json
        $sppkgFiles = @($allFiles | Where-Object { $_.Name -like '*.sppkg' })
        
        $script:Summary.FilesFound = $sppkgFiles.Count

        if ($sppkgFiles.Count -eq 0) {
            Write-Host "No .sppkg files found in app catalog" -ForegroundColor Yellow
            return
        }

        Write-Host "Found $($sppkgFiles.Count) .sppkg file(s). Starting download..." -ForegroundColor Cyan

        foreach ($file in $sppkgFiles) {
            if ($PSCmdlet.ShouldProcess($file.Name, 'Download .sppkg file')) {
                try {
                    $localPath = Join-Path -Path $OutputPath -ChildPath $file.Name
                    
                    m365 spo file get --webUrl $AppCatalogUrl --url $file.ServerRelativeUrl --asFile --path $localPath 2>&1 | Out-Null
                    
                    if ($LASTEXITCODE -ne 0) {
                        throw "CLI returned error"
                    }
                    
                    Write-Verbose "Downloaded: $($file.Name)"
                    $script:Summary.Downloaded++
                }
                catch {
                    Write-Warning "Failed to download '$($file.Name)': $($_.Exception.Message)"
                    $script:Summary.Failures++
                    continue
                }
            }
            else {
                $script:Summary.Downloaded++
            }
        }
    }
    catch {
        Write-Error "Script failed: $($_.Exception.Message)"
        throw
    }
}

end {
    Stop-Transcript

    Write-Host "`n===== Summary =====" -ForegroundColor Cyan
    Write-Host "App Catalog URL: $AppCatalogUrl" -ForegroundColor White
    Write-Host "Files Found: $($script:Summary.FilesFound)" -ForegroundColor White
    Write-Host "Downloaded: $($script:Summary.Downloaded)" -ForegroundColor Green
    
    $failureColor = if ($script:Summary.Failures -gt 0) { "Red" } else { "Green" }
    Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor $failureColor
    
    if (Test-Path -Path $OutputPath) {
        Write-Host "Output Path: $(Resolve-Path $OutputPath)" -ForegroundColor White
    }
}

# Example 1: Download all .sppkg files from tenant app catalog
# .\\Download-SPPKGs.ps1 -AppCatalogUrl "https://contoso.sharepoint.com/sites/apps"

# Example 2: Download to custom folder
# .\\Download-SPPKGs.ps1 -AppCatalogUrl "https://contoso.sharepoint.com/sites/apps" -OutputPath "C:\\Backups\\SPPKGs"

# Example 3: Test with WhatIf (see what would be downloaded)
# .\\Download-SPPKGs.ps1 -AppCatalogUrl "https://contoso.sharepoint.com/sites/apps" -WhatIf

# Example 4: Download with verbose output
# .\\Download-SPPKGs.ps1 -AppCatalogUrl "https://contoso.sharepoint.com/sites/apps" -OutputPath ".\\packages" -Verbose

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell

param (
  [Parameter(Mandatory = $true)]
  [string]$url,
  [Parameter(Mandatory = $true)]
  [string]$appCatalog,
  [Parameter(Mandatory = $true)]
  [string]$username,
  [Parameter(Mandatory = $true)]
  [string]$password
)

Clear-Host
Write-Progress -activity "Downloading packages..." -status "downloading" -PercentComplete 0

# Connect
$psw = ConvertTo-SecureString -String $password -AsPlainText -Force
$credentials = New-Object -TypeName System.Management.Automation.PSCredential -argumentlist $username, $psw
Connect-PnPOnline -Url $url -Credentials $credentials

Try {
  $list = Get-PnPList -Identity $appCatalog
  $folder = Get-PnPFolder -RelativeUrl $appCatalog
  $props = Get-PnPProperty -ClientObject $folder -Property Files
  $destinationfolder = ".\pkg"

  if (!(Test-Path -path $destinationfolder)) {
    $newItem = New-Item $destinationfolder -type directory
  }

  $item = Get-ChildItem -Path $destinationfolder -Include *.* -File -Recurse | ForEach-Object { $_.Delete() }
  $total = $folder.Files.Count

  For ($i = 0; $i -lt $total; $i++) {
    $file = $folder.Files[$i]
    $fileName = $file.Name
    $extn = [IO.Path]::GetExtension($file.Name)

    if ($extn -eq ".sppkg" ) {
      Write-Progress -activity "Downloading packages..." -status "downloading $fileName" -PercentComplete (($i / $total) * 100)
      $f = Get-PnPFile -ServerRelativeUrl $file.ServerRelativeUrl -Path $destinationfolder -FileName $file.Name -AsFile
    }
  }
}
Catch {
  Write-host -f Red "Error downloading packages:" $_.Exception.Message
}

Write-Host ("PACKAGES DOWNLOADED") -ForegroundColor Green
Disconnect-PnPOnline

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

---

## Contributors

| Author(s)                                 |
| ----------------------------------------- |
| [Adam Wójcik](https://github.com/Adam-it) |
| [Matteo Serpi](https://github.com/srpmtt) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-download-sppkgs" aria-hidden="true" />

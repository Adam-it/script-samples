

# Remove orphaned redirect sites

## Summary

Changing the URL of a site results in a new site type: a Redirect Site. However this redirect site does not get removed if you delete the newly renamed site. This could result in orphaned redirect site collections that redirect to nothing. This script provides you with an overview of all orphaned redirect sites and allows you to quickly delete them.

[!INCLUDE [Delete Warning](../../docfx/includes/DELETE-WARN.md)]
 
# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint Online tenant admin URL (must be HTTPS, e.g., https://contoso-admin.sharepoint.com)")]
    [ValidateScript({
        if ($_ -notmatch '^https://') {
            throw "TenantAdminUrl must use HTTPS protocol. Provided: $_"
        }
        $true
    })]
    [string]$TenantAdminUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Full path for CSV export file. Defaults to .\\OrphanedRedirectSites_<timestamp>.csv in current directory.")]
    [string]$OutputPath
)

begin {
    if (-not $OutputPath) {
        $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
        $OutputPath = Join-Path -Path (Get-Location) -ChildPath "OrphanedRedirectSites_$timestamp.csv"
        Write-Verbose "No output path specified. Using: $OutputPath"
    } else {
        $parentFolder = Split-Path -Path $OutputPath -Parent
        if (-not (Test-Path -Path $parentFolder)) {
            throw "Output folder does not exist: $parentFolder"
        }
    }

    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = $OutputPath -replace '\\.csv$', "_transcript_$timestamp.log"
    Start-Transcript -Path $transcriptPath

    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Orphaned Redirect Sites Removal" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host ""

    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to ensure Microsoft 365 login. Please run 'm365 login' manually."
    }
    Write-Host "[✓] Successfully authenticated to Microsoft 365" -ForegroundColor Green
    Write-Host ""

    $script:ReportCollection = [System.Collections.Generic.List[PSCustomObject]]::new()
    $script:Summary = @{
        TotalRedirects = 0
        OrphanedRemoved = 0
        ValidRedirects = 0
        Failures = 0
    }
}

process {
    Write-Host "Retrieving redirect sites from tenant..." -ForegroundColor Cyan
    Write-Verbose "Executing: m365 spo site list --filter \"Template eq 'RedirectSite#0'\" --output json"
    
    $sitesJson = m365 spo site list --filter "Template eq 'RedirectSite#0'" --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to retrieve redirect sites. Error: $sitesJson"
    }

    $redirectSites = @($sitesJson | ConvertFrom-Json)
    $script:Summary.TotalRedirects = $redirectSites.Count

    if ($redirectSites.Count -eq 0) {
        Write-Host "[✓] No redirect sites found in tenant." -ForegroundColor Green
        return
    }

    Write-Host "Found $($redirectSites.Count) redirect site(s). Analyzing redirect targets..." -ForegroundColor Yellow
    Write-Host ""

    $counter = 0
    foreach ($site in $redirectSites) {
        $counter++
        $siteUrl = $site.Url
        
        Write-Progress -Activity "Analyzing redirect sites" -Status "Processing $counter of $($redirectSites.Count): $siteUrl" -PercentComplete (($counter / $redirectSites.Count) * 100)
        Write-Verbose "Processing redirect site: $siteUrl"

        $reportItem = [PSCustomObject]@{
            RedirectSiteUrl = $siteUrl
            TargetUrl = ""
            TargetStatus = ""
            Action = ""
            ProcessedDate = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
            ErrorMessage = ""
        }

        try {
            $redirectResponse = Invoke-WebRequest -Uri $siteUrl -MaximumRedirection 0 -SkipHttpErrorCheck -ErrorAction Stop

            if ($redirectResponse.StatusCode -eq 308) {
                $targetUrl = $redirectResponse.Headers.Location
                $reportItem.TargetUrl = $targetUrl
                Write-Host "  Redirect: $siteUrl" -ForegroundColor Cyan
                Write-Host "    → Target: $targetUrl" -ForegroundColor Gray

                try {
                    $targetResponse = Invoke-WebRequest -Uri $targetUrl -SkipHttpErrorCheck -ErrorAction Stop
                    $reportItem.TargetStatus = $targetResponse.StatusCode

                    if ($targetResponse.StatusCode -eq 200) {
                        Write-Host "    [✓] Target exists (HTTP 200)" -ForegroundColor Green
                        $reportItem.Action = "Kept - Target exists"
                        $script:Summary.ValidRedirects++
                    }
                    elseif ($targetResponse.StatusCode -eq 404) {
                        Write-Host "    [×] Target not found (HTTP 404) - Orphaned redirect" -ForegroundColor Red

                        if ($PSCmdlet.ShouldProcess($siteUrl, 'Remove orphaned redirect site')) {
                            Write-Verbose "Executing: m365 spo site remove --url $siteUrl --force"
                            m365 spo site remove --url $siteUrl --force 2>&1 | Out-Null

                            if ($LASTEXITCODE -eq 0) {
                                Write-Host "    [✓] Removed orphaned redirect site" -ForegroundColor Yellow
                                $reportItem.Action = "Removed - Orphaned (moved to recycle bin)"
                                $script:Summary.OrphanedRemoved++
                            } else {
                                Write-Warning "    Failed to remove redirect site: $siteUrl"
                                $reportItem.Action = "Failed to remove"
                                $reportItem.ErrorMessage = "CLI command failed"
                                $script:Summary.Failures++
                            }
                        } else {
                            $reportItem.Action = "WhatIf - Would remove orphaned redirect"
                        }
                    }
                    else {
                        Write-Host "    [i] Target returned HTTP $($targetResponse.StatusCode)" -ForegroundColor Yellow
                        $reportItem.Action = "Kept - Unexpected status code"
                        $script:Summary.ValidRedirects++
                    }
                }
                catch {
                    Write-Warning "    Failed to check target URL: $($_.Exception.Message)"
                    $reportItem.TargetStatus = "Error"
                    $reportItem.Action = "Kept - Could not verify target"
                    $reportItem.ErrorMessage = $_.Exception.Message
                    $script:Summary.Failures++
                }
            }
            else {
                Write-Host "  [i] Site returned HTTP $($redirectResponse.StatusCode) (not a redirect)" -ForegroundColor Yellow
                $reportItem.TargetStatus = "Not a redirect (HTTP $($redirectResponse.StatusCode))"
                $reportItem.Action = "Skipped - Not a redirect"
            }
        }
        catch {
            Write-Warning "  Failed to process redirect site: $($_.Exception.Message)"
            $reportItem.TargetStatus = "Error"
            $reportItem.Action = "Skipped - Error"
            $reportItem.ErrorMessage = $_.Exception.Message
            $script:Summary.Failures++
        }

        $script:ReportCollection.Add($reportItem)
        Write-Host ""
    }

    Write-Progress -Activity "Analyzing redirect sites" -Completed
}

end {
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Total redirect sites found: $($script:Summary.TotalRedirects)" -ForegroundColor White
    Write-Host "Valid redirects (kept): $($script:Summary.ValidRedirects)" -ForegroundColor Green
    Write-Host "Orphaned redirects removed: $($script:Summary.OrphanedRemoved)" -ForegroundColor Yellow
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Green
    }
    Write-Host ""

    if ($script:ReportCollection.Count -gt 0) {
        Write-Host "Exporting report to: $OutputPath" -ForegroundColor Cyan
        $script:ReportCollection | Sort-Object 'RedirectSiteUrl' | Export-Csv -Path $OutputPath -NoTypeInformation -Force
        Write-Host "[✓] Report exported successfully" -ForegroundColor Green
    } else {
        Write-Host "[i] No data to export" -ForegroundColor Yellow
    }

    Stop-Transcript
    Write-Host "[✓] Transcript saved to: $transcriptPath" -ForegroundColor Green
}

# Example: Check for orphaned redirect sites in WhatIf mode
# .\\Remove-OrphanedRedirectSites.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -WhatIf

# Example: Remove orphaned redirect sites with default output path
# .\\Remove-OrphanedRedirectSites.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com"

# Example: Remove orphaned redirect sites with custom output path
# .\\Remove-OrphanedRedirectSites.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\\Reports\\OrphanedSites.csv"

# Example: Run with verbose output
# .\\Remove-OrphanedRedirectSites.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)
```powershell
$tenantAdminUrl = "https://contoso-admin.sharepoint.com" # Change to your tenant

Connect-PnPOnline -Url $tenantAdminUrl -Interactive

$sites = Get-PnPTenantSite -Template "RedirectSite#0"

$sites | ForEach-Object {
  Write-Host -f Green "Processing redirect site: " $_.Url
  $siteUrl = $_.Url

  $redirectSite = Invoke-WebRequest -Uri $_.Url -MaximumRedirection 0 -SkipHttpErrorCheck #Requires PowerShell 7 for -SkipHttpErrorCheck parameter
  $body = $null
  $siteUrl = $_.Url

  if($redirectSite.StatusCode -eq 308) {
    Try {
      [string]$newUrl = $redirectSite.Headers.Location;
      Write-Host -f Green " Redirects to: " $newUrl
      $body = Invoke-WebRequest -Uri $newUrl -SkipHttpErrorCheck #Requires PowerShell 7 for -SkipHttpErrorCheck parameter
    }
    Catch{
     Write-Host $_.Exception
    }
    Finally {
      If($body.StatusCode -eq "200"){
       Write-host -f Yellow "  Target location still exists"
      }
      If($body.StatusCode -eq "404"){
        Write-Host -f Red "  Target location no longer exists, should be removed"
        Remove-PnPTenantSite -Url $siteUrl -Force
      }
    }
  }
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it)|
| [Leon Armston](https://github.com/LeonArmston)|


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-remove-orphaned-redirect-sites" aria-hidden="true" />


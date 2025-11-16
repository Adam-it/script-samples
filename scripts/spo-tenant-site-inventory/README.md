

# List of active sites in Tenant with Admins and storage used

## Summary

This script provides you the list of active sites in your tenant with their administrator and usage in MB.


# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
function Get-SpoTenantSiteInventory {
    [CmdletBinding(SupportsShouldProcess = $false)]
    param(
        [Parameter(Mandatory = $false, HelpMessage = "OData filter applied to the list of sites (for example: StorageUsedMB gt 1000)")]
        [ValidateNotNullOrEmpty()]
        [string]$Filter,

        [Parameter(Mandatory = $false, HelpMessage = "Optional output CSV path; defaults to timestamped file in current folder")]
        [ValidateNotNullOrEmpty()]
        [string]$ReportPath
    )

    begin {
        Write-Host "Ensuring CLI session is authenticated..."
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to authenticate with CLI for Microsoft 365. Details: $loginOutput"
        }

        if (-not $ReportPath) {
            $timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
            $ReportPath = Join-Path -Path (Get-Location) -ChildPath "spo-site-inventory-$timestamp.csv"
        }

        $Summary = [pscustomobject]@{
            SitesEvaluated = 0
            SitesExported = 0
            AdminLookupFailures = 0
            ReportPath = $ReportPath
        }

        $ReportRows = [System.Collections.Generic.List[psobject]]::new()
    }

    process {
        $siteCommand = @(
            'spo', 'site', 'list',
            '--includeDetail',
            '--output', 'json'
        )

        if ($Filter) {
            $siteCommand += @('--filter', $Filter)
        }

        $siteCommand += @('--query', '[].{Title:Title,Url:Url,Template:Template,StorageUsageMB:StorageUsageMB,StorageQuotaMB:StorageQuotaMB,LastContentModifiedDate:LastContentModifiedDate}')

        $sitesJson = m365 @siteCommand 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list sites. CLI output: $sitesJson"
        }

        try {
            $sites = $sitesJson | ConvertFrom-Json
        }
        catch {
            throw "Unable to parse site listing response. $_"
        }
        if (-not $sites) {
            Write-Warning "No SharePoint sites matched the specified filter."
            return
        }

        foreach ($site in $sites) {
            $Summary.SitesEvaluated++

            $adminsJson = m365 spo site admin list --siteUrl $site.Url --output json --query '[].UserPrincipalName' 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve administrators for '$($site.Url)'. CLI output: $adminsJson"
                $Summary.AdminLookupFailures++
                continue
            }

            try {
                $admins = $adminsJson | ConvertFrom-Json
            }
            catch {
                Write-Warning "Unable to parse administrators for '$($site.Url)'. $_"
                $Summary.AdminLookupFailures++
                continue
            }

            $storageValue = if ($site.PSObject.Properties['StorageUsageMB']) {
                [double]$site.StorageUsageMB
            } elseif ($site.PSObject.Properties['StorageUsedMB']) {
                [double]$site.StorageUsedMB
            } elseif ($site.PSObject.Properties['StorageUsage']) {
                [double]$site.StorageUsage
            } else {
                0
            }

            $storageUsedMb = [math]::Round($storageValue, 2)
            $storageQuotaMb = if ($site.PSObject.Properties['StorageQuotaMB']) {
                [math]::Round([double]$site.StorageQuotaMB, 2)
            } else { $null }

            $ReportRows.Add([pscustomobject]@{
                Title = $site.Title
                Url = $site.Url
                Template = $site.Template
                StorageUsedMB = $storageUsedMb
                StorageQuotaMB = $storageQuotaMb
                LastContentModified = $site.LastContentModifiedDate
                Administrators = ($admins -join '; ')
            }) | Out-Null

            $Summary.SitesExported++
        }
    }

    end {
        if ($ReportRows.Count -gt 0) {
            $directory = Split-Path -Path $Summary.ReportPath -Parent
            if ([string]::IsNullOrWhiteSpace($directory)) {
                $directory = (Get-Location).Path
            }

            if (-not (Test-Path -Path $directory)) {
                try {
                    New-Item -ItemType Directory -Path $directory -Force | Out-Null
                }
                catch {
                    Write-Warning "Failed to create directory '$directory'. $_"
                }
            }

            try {
                $ReportRows | Export-Csv -Path $Summary.ReportPath -NoTypeInformation -Encoding UTF8
                Write-Host "Report saved to $($Summary.ReportPath)" -ForegroundColor Cyan
            }
            catch {
                Write-Warning "Unable to write site inventory report. $_"
            }
        }
        else {
            Write-Host "No site data collected; report not generated." -ForegroundColor DarkGray
        }

        Write-Host "Summary:" -ForegroundColor Cyan
        Write-Host "  Sites evaluated: $($Summary.SitesEvaluated)"
        Write-Host "  Sites exported: $($Summary.SitesExported)"
        Write-Host "  Admin lookup failures: $($Summary.AdminLookupFailures)"
    }
}

# Example usage
Get-SpoTenantSiteInventory -ReportPath "./site-inventory.csv"
```

# [PnP PowerShell](#tab/pnpps)

```powershell

Connect-PnPOnline -Url "https://contoso-admin.sharepoint.com/" -Interactive
        
# Get all SharePoint sites
$sites = Get-PnPTenantSite

# Create an array to store the results
$results = @()

# Iterate through each site and gather required information
foreach ($site in $sites) {
    $siteUrl = $site.Url
    
    Connect-PnPOnline -Url $siteUrl -Interactive

    # Get site administrators
    $admins = Get-PnPSiteCollectionAdmin | Select-Object -ExpandProperty Title

    # Get site storage size
    $storageSize = Get-PnPTenantSite -Url $siteUrl | Select-Object -ExpandProperty StorageUsageCurrent

    # Create a custom object with the site information
    $siteInfo = [PSCustomObject]@{
        SiteUrl = $siteUrl
        Administrators = $admins -join ";"
        StorageSize = $storageSize.ToString() +" MB(s)"
    }

    # Add the site information to the results array
    $results += $siteInfo
}

# Output the results as a CSV file
$results | Export-Csv -Path "SiteInventory.csv" -NoTypeInformation

# Disconnect from SharePoint Online
Disconnect-PnPOnline

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

## Contributors

| Author(s) |
|-----------|
| [Diksha Bhura](https://github.com/Diksha-Bhura) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-tenant-site-inventory" aria-hidden="true" />

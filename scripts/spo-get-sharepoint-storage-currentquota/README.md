

# Get SharePoint Storage Usage Against Allocated Quota

## Summary

There is limited space allocated to the tenant. To ensure business continuity and smooth ongoing operation, it is imperative to keep an eye on its usage and take relevant actions suited to the circumstances. By default a SharePoint site is allocated 25 TB by default and OneDrive for Business site is allocated 1 TB by default. These settings can be amended manually to a different quota to control SharePoint site. The script will help to proactively monitor percent used against quota for each SharePoint site. This sample is available in both PnP PowerShell and CLI for Microsoft 365.

![Example Screenshot](assets/example.png)

### Prerequisites

- The user account that runs the script must have SharePoint Online tenant administrator access.

# [PnP PowerShell](#tab/pnpps)

```powershell
connect-pnpOnline -url https://contoso-admin.sharepoint.com/ -Interactive
$reportPath = "c:\temp\storage.csv"
Get-PnPTenantSite -IncludeOneDriveSites |  Sort-Object StorageUsageCurrent -Descending  | Select-Object Url, @{Name='StorageUsageCurrent (GB)'; Expression={$_.StorageUsageCurrent / 1024}}, @{Name='StorageQuota (GB)'; Expression={$_.StorageQuota / 1024}}, @{Name='% Used'; Expression={'{0:P2}' -f ($_.StorageUsageCurrent / $_.StorageQuota)}} | Select-Object -First 10| export-csv  $reportPath -notypeinformation

# Omit `Select-Object -First 10` if all sites need to be monitored. 

# omit the parameter -IncludeOneDriveSites to exclude OneDrive sites 
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(HelpMessage = "Path where the CSV file will be saved")]
    [string]$OutputPath = "storage.csv",
    
    [Parameter(HelpMessage = "Include OneDrive sites in the report")]
    [switch]$IncludeOneDriveSites,
    
    [Parameter(HelpMessage = "OData filter to apply when retrieving sites (e.g., 'Template eq GROUP#0' or 'Url -like project')")]
    [string]$Filter,
    
    [Parameter(HelpMessage = "Export results to CSV file")]
    [switch]$ExportToCsv
)

begin {
    # Validate output path if exporting to CSV
    if ($ExportToCsv) {
        $outputDir = Split-Path -Path $OutputPath -Parent
        if ($outputDir -and -not (Test-Path -Path $outputDir)) {
            try {
                New-Item -Path $outputDir -ItemType Directory -Force | Out-Null
                Write-Verbose "Created output directory: $outputDir"
            } catch {
                throw "Failed to create output directory '$outputDir': $_"
            }
        }
    }
    
    # Verify CLI for Microsoft 365 login
    Write-Verbose "Verifying CLI for Microsoft 365 login status..."
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' first."
    }
    Write-Verbose "Successfully verified CLI for Microsoft 365 connection."
    
    # Initialize summary tracking
    $script:Summary = @{
        SitesFound = 0
        SitesExported = 0
        Failures = 0
    }
}

process {
    Write-Host "Retrieving site collections from tenant..."
    
    # Build CLI command with optional parameters
    if ($IncludeOneDriveSites) {
        Write-Verbose "Including OneDrive sites in the query..."
        if ($Filter) {
            Write-Warning "OneDrive sites cannot be combined with filters. Ignoring filter parameter."
            $sitesJson = m365 spo site list --withOneDriveSites --output json 2>&1
        } else {
            $sitesJson = m365 spo site list --withOneDriveSites --output json 2>&1
        }
    } else {
        if ($Filter) {
            Write-Verbose "Applying filter: $Filter"
            $sitesJson = m365 spo site list --filter $Filter --output json 2>&1
        } else {
            $sitesJson = m365 spo site list --output json 2>&1
        }
    }
    
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve site collections. CLI output: $sitesJson"
    }
    
    $sites = @($sitesJson | ConvertFrom-Json)
    $script:Summary.SitesFound = $sites.Count
    Write-Host "Found $($sites.Count) site collections."
    
    # Process sites and calculate storage metrics
    $count = 0
    $allSites = foreach ($site in $sites) {
        $count++
        Write-Verbose "Processing site $count of $($sites.Count): $($site.Url)"
        
        try {
            # Convert storage from MB to GB and calculate percentage
            $storageUsageGB = [math]::Round($site.StorageUsage / 1024, 2)
            $storageQuotaGB = [math]::Round($site.StorageMaximumLevel / 1024, 2)
            
            # Handle division by zero for percentage calculation
            if ($site.StorageMaximumLevel -gt 0) {
                $percentUsed = "{0:P2}" -f ($site.StorageUsage / $site.StorageMaximumLevel)
            } else {
                $percentUsed = "N/A"
            }
            
            [PSCustomObject]@{
                Url = $site.Url
                'StorageUsageCurrent (GB)' = $storageUsageGB
                'StorageQuota (GB)' = $storageQuotaGB
                '% Used' = $percentUsed
            }
            
            $script:Summary.SitesExported++
            
        } catch {
            Write-Warning "Failed to process site '$($site.Url)': $_"
            $script:Summary.Failures++
        }
    }
    
    # Sort by storage usage descending
    Write-Verbose "Sorting sites by storage usage..."
    $allSites = $allSites | Sort-Object 'StorageUsageCurrent (GB)' -Descending
}

end {
    # Display summary
    Write-Host "`n=== Export Summary ===" -ForegroundColor Cyan
    Write-Host "Sites Found: $($script:Summary.SitesFound)"
    Write-Host "Sites Exported: $($script:Summary.SitesExported)"
    Write-Host "Failures: $($script:Summary.Failures)"
    
    if ($ExportToCsv) {
        # Export to CSV with pipe delimiter (matching PnP script)
        if ($PSCmdlet.ShouldProcess($OutputPath, "Export storage quota information to CSV")) {
            try {
                $allSites | Export-Csv -Path $OutputPath -Encoding UTF8 -Force -Delimiter "|" -NoTypeInformation
                Write-Host "`nResults exported to: $OutputPath" -ForegroundColor Green
                Write-Host "Total sites in report: $($allSites.Count)" -ForegroundColor Green
            } catch {
                Write-Error "Failed to export CSV: $_"
            }
        }
    } else {
        # Display in terminal
        Write-Host "`n=== Storage Quota Information ===" -ForegroundColor Cyan
        $allSites | Format-Table -AutoSize
    }
}

# Display all SharePoint sites storage in terminal with verbose output
.\Get-SPOStorageQuota.ps1 -Verbose

# Export all SharePoint and OneDrive sites to CSV with verbose output
.\Get-SPOStorageQuota.ps1 -IncludeOneDriveSites -ExportToCsv -OutputPath "C:\\Reports\\storage.csv" -Verbose

# Export only Teams sites (GROUP#0 template) storage to CSV
.\Get-SPOStorageQuota.ps1 -Filter "Template eq 'GROUP#0'" -ExportToCsv -Verbose

# Export sites with 'project' in URL to CSV
.\Get-SPOStorageQuota.ps1 -Filter "Url -like 'project'" -ExportToCsv -OutputPath "project-sites-storage.csv" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

Sample first appeared on [SharePoint Storage Monitoring Against Allocated Quota using PowerShell](https://reshmeeauckloo.com/posts/PowerShell-SharePoint-Storage-Reporting/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it)|


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-sharepoint-storage-currentquota" aria-hidden="true" />

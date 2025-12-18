

# Sample showing how to use extract basic site collection properties to a CSV file

## Summary

Often we will have to provide a report containing various Site Collection properties to stakeholders which do not have access to the SharePoint Admin Center. This sample shows an export of a few basic properties but it can be expanded without too much hassle. This sample is available in both PnP PowerShell and CLI for Microsoft 365.

## Implementation

- Open VS Code
- Create a new file
- Copy the code below,
- Change the variables to target to your environment
- Run the script.
 
## Screenshot of Output 

![Example Screenshot](assets/preview.png)

# [PnP PowerShell](#tab/pnpps)
```powershell

$SharePointAdminUrl = "https://yourtenant-admin.sharepoint.com"

#connect to the admin site using one of the many options provided by Connect-PnPOnline
#Connect-PnPOnline -Url $SharePointAdminUrl -Interactive
#Connect-PnPOnline -Url $SharePointAdminUrl -ClientId XXXX -Tenant 'contoso.onmicrosoft.com' -Thumbprint YYYYY
Connect-PnPOnline -Url $SharePointAdminUrl -UseWebLogin

$allsites = Get-PnPTenantSite -Detailed 
Write-Host $allsites.Count

$Output = @()
$count = 0
foreach($s in $allsites)
{
    Write-Host " Working on item $count of $($allsites.Count))"
    $count++
    try 
    {
        $localconn = Connect-PnPOnline -Url $s.Url -Interactive -ReturnConnection
        
        $site = Get-PnPSite -Connection $localconn
        $web = Get-PnPWeb -Connection $localconn -ErrorAction stop

        $lastItemUserModifiedDate = Get-PnPProperty -ClientObject $web -Property "LastItemUserModifiedDate" -Connection $localconn
        $isHubsite = Get-PnPProperty -ClientObject $site -Property "IsHubSite" -Connection $localconn

        $myObject = [PSCustomObject]@{
            URL     = $s.Url
            GroupId = $s.GroupId
            LockState    = $s.LockState
            Template = $s.Template
            IsHubsite = $isHubSite
            ExternalAccess = $s.SharingCapability
            LastUserModifiedDate =  $lastItemUserModifiedDate
            StorageUsageCurrent = $s.StorageUsageCurrent 
            Error = "none"
        }        
    }
    catch 
    {
        $myObject = [PSCustomObject]@{
            URL     = $s.Url
            GroupId = $s.GroupId
            LockState    = $s.LockState
            Template = $s.Template
            IsHubsite = $isHubSite
            ExternalAccess = $s.SharingCapability
            StorageUsageCurrent = $s.StorageUsageCurrent
            Error = $_.Exception.message
        }
    
        
    }
    $Output+=($myObject)
}
$Output | Export-Csv  -Path c:\temp\sites.csv -Encoding utf8NoBOM -Force  -Delimiter "|"


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(HelpMessage = "Path where the CSV file will be saved")]
    [string]$OutputPath = "sites.csv",
    
    [Parameter(HelpMessage = "Optional OData filter to limit sites retrieved (e.g., \"Template eq 'STS#3'\" for Team Sites)")]
    [string]$Filter,
    
    [Parameter(HelpMessage = "Export results to CSV file")]
    [switch]$ExportToCsv
)

begin {
    # Validate output path if exporting to CSV
    if ($ExportToCsv) {
        Write-Verbose "Validating output path: $OutputPath"
        
        # Check if path is rooted (absolute) or relative
        $fullPath = if ([System.IO.Path]::IsPathRooted($OutputPath)) {
            $OutputPath
        } else {
            Join-Path -Path $PWD -ChildPath $OutputPath
        }
        
        # Get the directory path
        $directory = Split-Path -Path $fullPath -Parent
        
        # Validate directory exists or can be created
        if ($directory) {
            if (!(Test-Path -Path $directory)) {
                try {
                    Write-Verbose "Creating directory: $directory"
                    New-Item -ItemType Directory -Path $directory -Force -ErrorAction Stop | Out-Null
                } catch {
                    throw "Failed to create output directory '$directory': $_"
                }
            }
        }
        
        Write-Verbose "Output path validation successful."
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
    
    $allSites = @()
}

process {
    if ($Filter) {
        Write-Host "Retrieving site collections from tenant with filter: $Filter"
    } else {
        Write-Host "Retrieving all site collections from tenant..."
    }
    
    # Get sites with optional filter
    if ($Filter) {
        $sitesJson = m365 spo site list --filter $Filter --output json 2>&1
    } else {
        $sitesJson = m365 spo site list --output json 2>&1
    }
    
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve site collections. CLI output: $sitesJson"
    }
    
    $sites = @($sitesJson | ConvertFrom-Json)
    $script:Summary.SitesFound = $sites.Count
    Write-Host "Found $($sites.Count) site collections."
    
    # Process each site and map properties to match PnP output format
    $count = 0
    foreach ($site in $sites) {
        $count++
        Write-Verbose "Processing site $count of $($sites.Count): $($site.Url)"
        
        try {
            # Map CLI properties to PnP format
            $siteInfo = [PSCustomObject]@{
                URL = $site.Url
                GroupId = if ($site.GroupId) { $site.GroupId.Replace('/Guid(', '').Replace(')/', '') } else { '00000000-0000-0000-0000-000000000000' }
                LockState = $site.LockState
                Template = $site.Template
                IsHubsite = $site.IsHubSite
                ExternalAccess = $site.SharingCapability
                LastUserModifiedDate = $site.LastContentModifiedDate
                StorageUsageCurrent = $site.StorageUsage
                Error = "none"
            }
            
            $allSites += $siteInfo
            $script:Summary.SitesExported++
            
        } catch {
            Write-Warning "Failed to process site '$($site.Url)': $_"
            
            $siteInfo = [PSCustomObject]@{
                URL = $site.Url
                GroupId = if ($site.GroupId) { $site.GroupId.Replace('/Guid(', '').Replace(')/', '') } else { '00000000-0000-0000-0000-000000000000' }
                LockState = $site.LockState
                Template = $site.Template
                IsHubsite = $site.IsHubSite
                ExternalAccess = $site.SharingCapability
                LastUserModifiedDate = $null
                StorageUsageCurrent = $site.StorageUsage
                Error = $_.Exception.Message
            }
            
            $allSites += $siteInfo
            $script:Summary.Failures++
        }
    }
}

end {
    # Display summary
    Write-Host "`n=== Export Summary ===" -ForegroundColor Cyan
    Write-Host "Sites Found: $($script:Summary.SitesFound)"
    Write-Host "Sites Exported: $($script:Summary.SitesExported)"
    Write-Host "Failures: $($script:Summary.Failures)"
    
    if ($ExportToCsv) {
        # Export to CSV with pipe delimiter (matching PnP script)
        if ($PSCmdlet.ShouldProcess($OutputPath, "Export site collection info to CSV")) {
            try {
                $allSites | Export-Csv -Path $OutputPath -Encoding UTF8 -Force -Delimiter "|" -NoTypeInformation
                Write-Host "`nResults exported to: $OutputPath" -ForegroundColor Green
            } catch {
                Write-Error "Failed to export CSV: $_"
            }
        }
    } else {
        # Display in terminal
        Write-Host "`n=== Site Collection Information ===" -ForegroundColor Cyan
        $allSites | Format-Table -AutoSize
    }
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Kasper Larsen, Fellowmind|
| Adam Wójcik [@Adam-it](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-basic-sitecollection-info" aria-hidden="true" />

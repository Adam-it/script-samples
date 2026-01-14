

# Generate a csv report for a selection of site collections showing the time of the most recent update by any user

## Summary


![Example Screenshot](assets/example.png)


# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory=$true, HelpMessage="SharePoint admin center URL")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$AdminUrl,
    
    [Parameter(Mandatory=$false, HelpMessage="Usage report period (D7, D30, D90, D180)")]
    [ValidateSet('D7','D30','D90','D180')]
    [string]$UsagePeriod = 'D180',
    
    [Parameter(Mandatory=$false, HelpMessage="Site type filter (TeamSite, CommunicationSite, All)")]
    [ValidateSet('TeamSite','CommunicationSite','All')]
    [string]$SiteType = 'All',
    
    [Parameter(Mandatory=$false, HelpMessage="Output path for CSV report")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "SiteActivityReport_$timestamp.log"
    Start-Transcript -Path $transcriptPath | Out-Null

    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Cyan
    $loginOutput = m365 login --ensure 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to connect to Microsoft 365: $loginOutput"
    }

    Write-Host "Retrieving SharePoint site usage data from Microsoft Graph..." -ForegroundColor Cyan
    $usageJson = m365 spo report siteusagedetail --period $UsagePeriod --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve usage data: $usageJson"
    }

    $usageData = $usageJson | ConvertFrom-Json
    $usageLookup = @{}
    foreach ($item in $usageData) {
        $usageLookup[$item.'Site URL'] = $item
    }
    Write-Host "Retrieved usage data for $($usageData.Count) sites (period: $UsagePeriod)" -ForegroundColor Green

    $script:ReportCollection = @()
    $script:Summary = @{
        TotalSites = 0
        Processed = 0
        Failures = 0
    }

    Write-Host "Starting site activity report for admin site: $AdminUrl" -ForegroundColor Green
}

process {
    try {
        Write-Host "Retrieving site collections..." -ForegroundColor Cyan
        
        $siteListArgs = @('spo', 'site', 'list', '--output', 'json')
        if ($SiteType -ne 'All') {
            $siteListArgs += @('--type', $SiteType)
        }
        
        $sitesJson = m365 @siteListArgs 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sites: $sitesJson"
        }
        
        $sites = $sitesJson | ConvertFrom-Json
        $script:Summary.TotalSites = $sites.Count
        
        if ($sites.Count -eq 0) {
            Write-Host "No sites found" -ForegroundColor Yellow
            return
        }

        Write-Host "Found $($sites.Count) sites. Processing..." -ForegroundColor Cyan

        foreach ($site in $sites) {
            try {
                Write-Verbose "Processing site: $($site.Url)"
                
                $webJson = m365 spo web get --url $site.Url --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve web properties for '$($site.Url)': $webJson"
                    $script:Summary.Failures++
                    continue
                }

                $web = $webJson | ConvertFrom-Json
                $lastModified = $web.LastItemUserModifiedDate

                $usageInfo = $usageLookup[$site.Url]
                $lastActivityDate = if ($usageInfo -and $usageInfo.'Last Activity Date') {
                    $usageInfo.'Last Activity Date'
                } else {
                    "No usage data for period $UsagePeriod"
                }

                $script:ReportCollection += [PSCustomObject]@{
                    SiteUrl = $site.Url
                    SiteTitle = $site.Title
                    LastItemUserModifiedDate = $lastModified
                    LastActivityDateGraph = $lastActivityDate
                    UsagePeriod = $UsagePeriod
                }

                $script:Summary.Processed++
                Write-Verbose "  Last modified: $lastModified | Graph activity: $lastActivityDate"
            }
            catch {
                Write-Warning "Error processing site '$($site.Url)': $_"
                $script:Summary.Failures++
                continue
            }
        }
    }
    catch {
        Write-Error "Critical error in process block: $_"
        throw
    }
}

end {
    Stop-Transcript | Out-Null

    if ($script:ReportCollection.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "SiteActivityReport_$timestamp.csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "`nReport exported to: $csvPath" -ForegroundColor Green
    } else {
        Write-Host "`nNo data to export" -ForegroundColor Yellow
    }

    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "SITE ACTIVITY REPORT SUMMARY" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Total Sites Found   : $($script:Summary.TotalSites)" -ForegroundColor White
    Write-Host "Sites Processed     : $($script:Summary.Processed)" -ForegroundColor Green
    Write-Host "Failures            : $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "Usage Period        : $UsagePeriod" -ForegroundColor White
    Write-Host "========================================" -ForegroundColor Cyan
}

# Usage examples:
#
# Basic usage - all sites with D180 period:
# .\Generate-SiteActivityReport.ps1 -AdminUrl "https://contoso-admin.sharepoint.com"
#
# Team sites only with D30 period:
# .\Generate-SiteActivityReport.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -SiteType TeamSite -UsagePeriod D30
#
# With verbose output:
# .\Generate-SiteActivityReport.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -Verbose
#
# Custom output path:
# .\Generate-SiteActivityReport.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\Reports"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell

$ClientId = "xxxxxxxxx"
$TenantName = "contoso.onmicrosoft.com"
$thumbprint = "1234567890"
$SharePointAdminSiteURL = "https://contoso-admin.sharepoint.com"
$conn = Connect-PnPOnline -Url $adminSiteURL -ClientId $ClientId -Tenant $TenantName -Thumbprint $thumbprint -ReturnConnection
$UsageDays = 180

$accessToken = Get-PnPAccessToken -Connection $conn
$header = @{
    "Content-Type" = "application/json"
    Authorization = "Bearer $accessToken"
    }

#call graph getSharePointSiteUsageDetail
$GraphUrl = "https://graph.microsoft.com/v1.0/reports/getSharePointSiteUsageDetail(period='D$($UsageDays)')"
$UsageData = Invoke-RestMethod -Uri $GraphUrl -Method Get -Headers $header
$UsageDataAsObject = $UsageData | ConvertFrom-Csv

function GetUsageDatoForSiteCollection ($url)
{
    foreach($item in $UsageDataAsObject)
    {
        if($item."Site Url" -eq $url)
        {
            return $item
        }
    }
    return $null
}


$arrayList = New-Object System.Collections.ArrayList
$allsitecollections = Get-PnPTenantSite -Connection $conn
foreach($site in $allsitecollections)
{
    #get last item user modified date
    $localconn = Connect-PnPOnline -Url $site.Url -ClientId $ClientId -Tenant $TenantName -Thumbprint $thumbprint -ReturnConnection
    $token = Get-PnPAccessToken -Connection $localconn
    try {
        $web = Get-PnPWeb -Connection $localconn  -ErrorAction Stop
        $lastmod = Get-PnPProperty -ClientObject $web -Property LastItemUserModifiedDate -Connection $localconn   
        
        $object = New-Object PSObject
        $object | Add-Member -MemberType NoteProperty -Name "SiteUrl" -Value $site.Url
        $object | Add-Member -MemberType NoteProperty -Name "LastItemUserModifiedDate" -Value $lastmod.Date
        
        
        $UsageDataForSiteCollection = getUsageDatoForSiteCollection $site.Url
        If($UsageDataForSiteCollection -eq $null -or $UsageDataForSiteCollection.'Last Activity Date' -eq "")
        {
            Write-Host "No usage data for site collection $($site.Url) for the last $UsageDays days"
            $object | Add-Member -MemberType NoteProperty -Name "Last Activity Date (Graph)" -Value "No usage data for the last $UsageDays days" 
        }
        else 
        {
            $object | Add-Member -MemberType NoteProperty -Name "Last Activity Date (Graph)" -Value $UsageDataForSiteCollection.'Last Activity Date'
        }
        $arrayList.add( $object) | Out-Null
    }
    catch 
    {
        Write-Host $_.Exception.Message
    }
}
$arrayList | Export-Csv -Path "C:\temp\LastActivity.csv"  -Force -Delimiter "|" -Encoding utf8


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-most-recent-update-report" aria-hidden="true" />

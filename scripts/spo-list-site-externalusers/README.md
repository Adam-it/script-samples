

# List all external users in all site collections

## Summary

This script helps you to list all external users in all SharePoint Online sites. It provides insights in who the users are, and if available who they where invited by.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint Online tenant admin URL (must be HTTPS, e.g., https://contoso-admin.sharepoint.com)")]
    [ValidateScript({
        if ($_ -notmatch '^https://') {
            throw "TenantAdminUrl must use HTTPS protocol. Provided: $_"
        }
        $true
    })]
    [string]$TenantAdminUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Full path for CSV export file. Defaults to .\\ExternalUsers_<timestamp>.csv in current directory.")]
    [string]$OutputPath,

    [Parameter(Mandatory = $false, HelpMessage = "OData filter for site retrieval. Default: 'Url -like '/sites/'' (only /sites/* sites). Use empty string for all sites.")]
    [string]$SiteFilter = "Url -like '/sites/'"
)

begin {
    if (-not $OutputPath) {
        $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
        $OutputPath = Join-Path -Path (Get-Location) -ChildPath "ExternalUsers_$timestamp.csv"
        Write-Verbose "No output path specified. Using: $OutputPath"
    }

    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = $OutputPath -replace '\\.csv$', "_transcript_$timestamp.log"
    Start-Transcript -Path $transcriptPath

    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "External Users Report - CLI for M365" -ForegroundColor Cyan
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

    $parentFolder = Split-Path -Path $OutputPath -Parent
    if (-not (Test-Path -Path $parentFolder)) {
        Stop-Transcript
        throw "Output folder does not exist: $parentFolder"
    }

    $script:ReportCollection = [System.Collections.Generic.List[PSCustomObject]]::new()
    $script:Summary = @{
        SitesProcessed = 0
        SitesFailed = 0
        ExternalUsersFound = 0
    }
}

process {
    Write-Host "Retrieving sites from tenant..." -ForegroundColor Yellow
    Write-Verbose "Using filter: $SiteFilter"

    $siteListArgs = @('spo', 'site', 'list', '--output', 'json')
    if ($SiteFilter) {
        $siteListArgs += @('--filter', $SiteFilter)
    }
    $sitesJson = m365 @siteListArgs 2>&1

    if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve sites. CLI: $sitesJson"
        return
    }

    $sites = @($sitesJson | ConvertFrom-Json)
    $siteCount = $sites.Count
    Write-Host "[✓] Found $siteCount sites to process" -ForegroundColor Green
    Write-Host ""

    if ($siteCount -eq 0) {
        Write-Host "No sites found matching filter. Exiting." -ForegroundColor Yellow
        return
    }

    $siteCounter = 0
    foreach ($site in $sites) {
        $siteCounter++
        $percentComplete = [math]::Round(($siteCounter / $siteCount) * 100)
        Write-Progress -Activity "Processing sites for external users" -Status "Site $siteCounter of $siteCount" -PercentComplete $percentComplete
        Write-Verbose "[$siteCounter/$siteCount] Processing: $($site.Url)"

        try {
            $position = 0
            $pageSize = 50
            $siteExternalUserCount = 0

            do {
                $externalUsersJson = m365 spo externaluser list --siteUrl $($site.Url) --pageSize $pageSize --position $position --output json 2>&1
                
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve external users for site: $($site.Url). CLI: $externalUsersJson"
                    $script:Summary.SitesFailed++
                    break
                }

                $externalUsers = @($externalUsersJson | ConvertFrom-Json)
                $externalUserCount = $externalUsers.Count

                foreach ($user in $externalUsers) {
                    $reportItem = [PSCustomObject]@{
                        'Site Name' = $site.Title
                        'Site URL' = $site.Url
                        'Display Name' = $user.Title
                        'Email' = $user.Email
                        'Login Name' = $user.LoginName
                        'User Principal Name' = $user.UserPrincipalName
                        'Is Share By Email Guest' = $user.IsShareByEmailGuestUser
                        'Is Email Authentication Guest' = $user.IsEmailAuthenticationGuestUser
                        'Is Site Admin' = $user.IsSiteAdmin
                        'Expiration' = $user.Expiration
                    }
                    $script:ReportCollection.Add($reportItem)
                    $siteExternalUserCount++
                }

                $position += $pageSize

            } while ($externalUserCount -eq $pageSize)

            $script:Summary.SitesProcessed++
            $script:Summary.ExternalUsersFound += $siteExternalUserCount

            if ($siteExternalUserCount -gt 0) {
                Write-Verbose "  [✓] Found $siteExternalUserCount external user(s)"
            }

        } catch {
            Write-Warning "Error processing site $($site.Url): $($_.Exception.Message)"
            $script:Summary.SitesFailed++
            continue
        }
    }

    Write-Progress -Activity "Processing sites for external users" -Completed
}

end {
    Write-Host ""
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Sites processed:        $($script:Summary.SitesProcessed)" -ForegroundColor $(if ($script:Summary.SitesProcessed -gt 0) { 'Green' } else { 'Yellow' })
    Write-Host "Sites failed:           $($script:Summary.SitesFailed)" -ForegroundColor $(if ($script:Summary.SitesFailed -gt 0) { 'Red' } else { 'Green' })
    Write-Host "External users found:   $($script:Summary.ExternalUsersFound)" -ForegroundColor $(if ($script:Summary.ExternalUsersFound -gt 0) { 'Green' } else { 'Yellow' })
    Write-Host ""

    if ($script:ReportCollection.Count -gt 0) {
        $script:ReportCollection | Sort-Object 'Site Name', 'Display Name' | Export-Csv -Path $OutputPath -NoTypeInformation -Force
        Write-Host "[✓] Report exported to: $OutputPath" -ForegroundColor Green
    } else {
        Write-Host "[!] No external users found. CSV not created." -ForegroundColor Yellow
    }

    Write-Host "[✓] Transcript saved to: $transcriptPath" -ForegroundColor Green
    Stop-Transcript
}

# Usage examples:

# Basic usage with default output path (current directory)
# .\Get-ExternalUsers.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\Reports\ExternalUsers.csv"

# Scan all sites (including OneDrive, root site)
# .\Get-ExternalUsers.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\Reports\ExternalUsers.csv" -SiteFilter ""

# Run with verbose output for detailed progress
# .\Get-ExternalUsers.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\Reports\ExternalUsers.csv" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
 
# [SPO Management Shell](#tab/spoms-ps)

```powershell

$fileExportPath = "<PUTYOURPATHHERE.csv>"

Connect-SPOService https://<yourorg>-admin.sharepoint.com

$results = @()
Write-host "Retrieving all sites and check external users..."
$allSPOSites = Get-SPOSite -Limit ALL
$siteCount = $allSPOSites.Count

Write-Host "Processing $siteCount sites..."
#Loop through each site
$siteCounter = 0

foreach ($site in $allSPOSites) {
  $siteCounter++
  Write-Host "Processing $($site.Url)... ($siteCounter/$siteCount)"

  Write-host "Retrieving all external users ..."

  $users = Get-SPOExternalUser -SiteUrl $($site.Url)

  Write-host "  $($users.Count) external users ..." -ForegroundColor Yellow

  foreach ($user in $users) {
    
    $results = [pscustomobject][ordered]@{
      DisplayName = $user.DisplayName
      Email       = $user.Email
      WhenCreated = $user.WhenCreated
      Url         = $site.Url
    }

    $results | Export-Csv -Path $fileExportPath -NoTypeInformation -Append
  }
}


Write-Host "Completed."

```
[!INCLUDE [More about SPO Management Shell](../../docfx/includes/MORE-SPOMS.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell

#Global Variable Declaration
$AdminURL = "https://domain-admin.sharepoint.com/"
$TenantURL = "https://domain.SharePoint.com"
$UserName = "chandani@domain.onmicrosoft.com"
$Password = "********"
$SecureStringPwd = $Password | ConvertTo-SecureString -AsPlainText -Force 
$Credentials = New-Object System.Management.Automation.PSCredential -ArgumentList $UserName, $SecureStringPwd
$DateTime = "_{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$BasePath = "E:\Contribution\PnP-Scripts\GetExtenalUsers\Logs\"
$CSVPath = $BasePath + "\ExternalUsers" + $DateTime + ".csv"
$global:ExternalUsersData = @() 
Function LoginToAdminSite() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $Credentials)
    Write-Host "Connecting to Tenant Admin Site '$($AdminURL)'..." -ForegroundColor Yellow
    Connect-PnPOnline -Url $AdminURL -Credentials $Credentials
    Write-Host "Connection Successfull to Tenant Admin Site :'$($AdminURL)'" -ForegroundColor Green
}
Function ConnectToSPSite() {
    try {
        $SiteCollection = Get-PnPTenantSite -Filter "Url -like '$TenantURL'" | Where { $_.SharingCapability -ne "Disabled" }
        foreach ($Site in $SiteCollection) {
            $SiteUrl = $Site.Url    
            Write-Host "Connecting to Site :'$($SiteUrl)'..." -ForegroundColor Yellow  
            Connect-PnPOnline -Url $SiteUrl -Credentials $Credentials
            Write-Host "Connection Successfull to site: '$($SiteUrl)'" -ForegroundColor Green              
            GetExternalUsers($SiteUrl)                        
        }
        ExportData       
    }
    catch {
        Write-Host "Error in connecting to Site:'$($SiteUrl)'" $_.Exception.Message -ForegroundColor Red               
    } 
}
Function GetExternalUsers($siteUrl) {
    try {
        $ExternalUsers = Get-PnPUser | Where { $_.LoginName -like "*#ext#*" -or $_.LoginName -like "*urn:spo:guest*" }   
        Write-host "Found '$($ExternalUsers.count)' External users" -ForegroundColor Gray
        ForEach ($User in $ExternalUsers) {
            $global:ExternalUsersData += New-Object PSObject -Property ([ordered]@{
                    SiteName  = $site.Title
                    SiteURL   = $SiteUrl
                    UserName  = $User.Title
                    Email     = $User.Email
                    LoginName = $User.LoginName
                })
        }          
    }
    catch {
        Write-Host "Error in getting external users :'$($siteUrl)'" $_.Exception.Message -ForegroundColor Red                 
    }        
}

Function ExportData {
    Write-Host "Exporting to CSV" -ForegroundColor Yellow           
    $global:ExternalUsersData | Export-Csv -Path $CSVPath -NoTypeInformation -Append
    Write-Host "Exported Successfully!" -ForegroundColor Green 
}

Function StartProcessing {   
    LoginToAdminSite($AdminURL) 
    ConnectToSPSite
}

StartProcessing

```
***

## Source Credit

Sample first appeared on [List all external users in all site collections | CLI for Microsoft 365](https://pnp.github.io/cli-microsoft365/sample-scripts/spo/list-site-externalusers/)

## Contributors

| Author(s) |
|-----------|
| Adam Wójcik |
| Paul Bullock |
| Chandani Prajapati |
| Martin Lingstuyl |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-list-site-externalusers" aria-hidden="true" />

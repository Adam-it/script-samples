

# List external users across all sites and in what site groups they are

## Summary
This script demonstrates how to audit external users across all SharePoint Online sites and identify which site groups they belong to. Available in both CLI for Microsoft 365 and PnP PowerShell versions, it provides comprehensive reporting on external user group memberships with detailed CSV output.
This script shows how you can check if external users are added to site groups. It will show all external users across all site collections and the site groups they where added to.

![Example Screenshot](assets/example.png)
 
# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $false, HelpMessage = "Path where the CSV report will be saved")]
    [ValidateScript({
        if (Test-Path -Path $_ -IsValid) { $true }
        else { throw "Invalid path: $_" }
    })]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyy-MM-dd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "spo-external-users-transcript-$timestamp.log"
    Start-Transcript -Path $transcriptPath

    Write-Host "Starting external users in site groups report..." -ForegroundColor Cyan

    Write-Verbose "Ensuring CLI for Microsoft 365 connection..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to connect to Microsoft 365. Please run 'm365 login' manually."
    }

    $script:Summary = @{
        SitesProcessed = 0
        ExternalUsers = 0
        GroupMemberships = 0
        Failures = 0
    }
    
    $script:ReportCollection = [System.Collections.ArrayList]::new()
}

process {
    Write-Host "`nRetrieving all sites..." -ForegroundColor Green
    
    try {
        $sitesJson = m365 spo site list --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sites list"
        }
        $sites = $sitesJson | ConvertFrom-Json
        $siteCount = $sites.Count
        
        Write-Host "Found $siteCount sites. Processing..." -ForegroundColor Cyan
    }
    catch {
        Write-Warning "Failed to retrieve sites: $_"
        $script:Summary.Failures++
        return
    }

    $siteCounter = 0

    foreach ($site in $sites) {
        $siteCounter++
        Write-Verbose "[$siteCounter/$siteCount] Processing: $($site.Url)"
        
        try {
            $spoAccessToken = m365 util accesstoken get --resource sharepoint --new
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to get access token for $($site.Url)"
            }
            
            $uri = "$($site.Url)/_api/web/siteusers?`$filter=IsShareByEmailGuestUser eq true&`$expand=Groups&`$select=Title,LoginName,Email,Groups/LoginName"
            $headers = @{
                Authorization = "Bearer $spoAccessToken"
                Accept = "application/json;odata=nometadata"
            }
            
            $response = Invoke-WebRequest -Uri $uri -Method Get -Headers $headers -ErrorAction Stop
            $users = ($response.Content | ConvertFrom-Json).value
            
            if ($users.Count -gt 0) {
                Write-Host "  Found $($users.Count) external user(s) in $($site.Url)" -ForegroundColor Yellow
                $script:Summary.ExternalUsers += $users.Count
            }
            
            foreach ($user in $users) {
                foreach ($group in $user.Groups) {
                    $script:ReportCollection.Add([PSCustomObject]@{
                        SiteUrl = $site.Url
                        UserTitle = $user.Title
                        UserEmail = $user.Email
                        UserLoginName = $user.LoginName
                        GroupLoginName = $group.LoginName
                        Timestamp = (Get-Date -Format "yyyy-MM-dd HH:mm:ss")
                    }) | Out-Null
                    $script:Summary.GroupMemberships++
                }
            }
            
            $script:Summary.SitesProcessed++
        }
        catch {
            Write-Warning "Failed to process site '$($site.Url)': $_"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Sites Processed: $($Summary.SitesProcessed)" -ForegroundColor Green
    Write-Host "External Users Found: $($Summary.ExternalUsers)" -ForegroundColor Green
    Write-Host "Group Memberships: $($Summary.GroupMemberships)" -ForegroundColor Green
    Write-Host "Failures: $($Summary.Failures)" -ForegroundColor $(if ($Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "============================`n" -ForegroundColor Cyan

    if ($ReportCollection.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "spo-external-users-in-groups-$timestamp.csv"
        $ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "Report exported to: $csvPath" -ForegroundColor Green
    }
    else {
        Write-Host "No external users found in site groups." -ForegroundColor Yellow
    }

    Stop-Transcript
}

# Usage examples:
#
# Example 1: Run with default output path (current directory)
# .\Get-ExternalUsersInGroups.ps1
#
# Example 2: Specify custom output path
# .\Get-ExternalUsersInGroups.ps1 -OutputPath "C:\Reports"
#
# Example 3: Run with verbose output to see detailed processing
# .\Get-ExternalUsersInGroups.ps1 -Verbose
#
# Example 4: Validate parameters without executing (test path validation)
# .\Get-ExternalUsersInGroups.ps1 -OutputPath "InvalidPath" -WhatIf
```

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell

Connect-PnPOnline "https://contoso-admin.sharepoint.com" -Interactive

Write-Host "Retrieving all sites and check external users..." -ForegroundColor Green

$sites = Get-PnPTenantSite
$siteCount = $sites.Count
$siteCounter = 0
$results = [System.Collections.ArrayList]::new()

Write-Host "Processing $siteCount sites..."

foreach($site in $sites) {
  $siteCounter++
  Write-Host "$siteCounter/$siteCount - Get external users in site groups for $($site.Url)..." -ForegroundColor Green
  
  Connect-PnPOnline -Url $site.Url -Interactive
    
  $users = (Invoke-PnPSPRestMethod -Method Get -Url "$($site.Url)/_api/web/siteusers?`$filter=IsShareByEmailGuestUser eq true&`$expand=Groups&`$select=Title,LoginName,Email,Groups/LoginName" -ContentType "application/json;odata=nometadata" -Raw -ErrorAction Ignore | ConvertFrom-Json)

  foreach($user in $users.value) {
    foreach($group in $user.Groups) {      
      $obj = [PSCustomObject][ordered]@{
          Title = $user.Title;
          Email = $user.Email;
          LoginName = $user.LoginName;
          Group = $group.LoginName;
      }
      $results.Add($obj) | Out-Null
    }
  }
}

Write-Host "Exporting list..." -ForegroundColor Green
$results | Export-Csv -Path "./pnp-external-users-in-sitegroups.csv" -NoTypeInformation

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Contributors

| Author(s) |
|-----------|
| Adam Wójcik |
| Martin Lingstuyl |
| Bart-Jan Dekker |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-list-site-externalusers-in-groups" aria-hidden="true" />

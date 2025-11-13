# SharePoint Online - Export SharePoint Alerts

## Summary

This PowerShell script scans your SharePoint Online environment and exports a list of all classic SharePoint Alerts configured by users across all site collections.

This is especially useful in preparation for the upcoming retirement of SharePoint Alerts. The script helps administrators audit alert usage, review who is using alerts and where, and prepare to transition to modern alternatives such as SharePoint Rules or Power Automate.

### SharePoint Alerts Retirement Timeline

Microsoft has announced the deprecation and removal of **classic SharePoint Alerts**. Use this script to assess current alert usage and prepare for a smooth transition.

#### Important Dates to Keep in Mind

- **July 2025** – Creation of new SharePoint alerts disabled for **new tenants**
- **September 2025** – Creation of new alerts disabled for **existing tenants**
- **October 2025** – Existing alerts will **expire after 30 days** (they can be manually re-enabled)
- **July 2026** – **SharePoint Alerts removed entirely**

After these dates, organisations should use modern alternatives like SharePoint Rules or Power Automate.

### Features

- Connects to all site collections in your tenant
- Retrieves classic alerts created by users
- Outputs a clean, structured CSV file with alert metadata
- Logs connection or permission errors to a separate file

### Sample Output

Below is an example of the exported `SharePointAlerts_Report.csv` when opened in Excel:

![Sample SharePoint Alerts Output](./assets/SharePointAlertAuditOutput.png)

***

# [PnP PowerShell](#tab/pnpps)
```powershell
# ============================================================
# SharePoint Alerts Export Script
# Preferred Auth: Certificate
# Alternate Auth: Interactive
# ============================================================

# === Configurable Variables ===
$clientId = "<your-client-id>"
$certPath = "<path-to-certificate-folder>"
$certName = "<your-certificate-name>.pfx"
$tenant = "<your-tenant-id>"

$basePath = "<your-output-directory>"  # Example: "C:\Reports"
$outputCsvName = "SharePointAlerts_Report.csv"
$errorLogName = "SharePointAlerts_Errors.csv"

$tenantAdminUrl = "https://<your-tenant>-admin.sharepoint.com"
$outputCsvPath = Join-Path -Path $basePath -ChildPath $outputCsvName
$errorLogPath = Join-Path -Path $basePath -ChildPath $errorLogName

# === Create output folder if it doesn't exist ===
if (-not (Test-Path -Path $basePath)) {
    New-Item -ItemType Directory -Path $basePath -Force | Out-Null
}

# === Arrays to collect results and errors ===
$alertResults = @()
$errorLog = @()

# === Connect to SharePoint Admin Center ===
# Preferred: Certificate-based authentication
Connect-PnPOnline -Url $tenantAdminUrl -ClientId $clientId -Tenant $tenant -CertificatePath "$certPath\$certName"

# Alternate: Interactive authentication
# Connect-PnPOnline -Url $tenantAdminUrl -Interactive

# === Retrieve all site collections ===
$sites = Get-PnPTenantSite

foreach ($site in $sites) {
    Write-Host "`nProcessing site: $($site.Url)" -ForegroundColor Cyan

    # === Connect to each site ===
    # Preferred: Certificate-based authentication
    Connect-PnPOnline -Url $site.Url -ClientId $clientId -Tenant $tenant -CertificatePath "$certPath\$certName"

    # Alternate (manual run): Interactive authentication
    # Connect-PnPOnline -Url $site.Url -Interactive

    try {
        $alerts = Get-PnPAlert -AllUsers -InformationAction SilentlyContinue
        if ($alerts) {
            Write-Host "Found $($alerts.Count) alerts in: $($site.Url)" -ForegroundColor Green
            foreach ($alert in $alerts) {
                $user = Get-PnPUser -Identity $alert.UserId
                $alertResults += [PSCustomObject]@{
                    SiteUrl          = $site.Url
                    AlertFrequency   = $alert.AlertFrequency
                    AlertType        = $alert.AlertType
                    AlertTime        = $alert.AlertTime
                    AlwaysNotify     = $alert.AlwaysNotify
                    DeliveryChannels = $alert.DeliveryChannels
                    EventType        = $alert.EventType
                    Filter           = $alert.Filter
                    ID               = $alert.ID
                    Status           = $alert.Status
                    Title            = $alert.Title
                    UserId           = $alert.UserId
                    UserName         = $user.Title
                    UserEmail        = $user.Email
                    DispFormUrl      = $alert.Properties["dispformurl"]
                }
            }
        } else {
            Write-Host "No alerts found." -ForegroundColor Yellow
        }
    }
    catch {
        $errorMessage = "Error retrieving alerts for site: $($site.Url)."
        Write-Host $errorMessage -ForegroundColor Red

        $errorLog += [PSCustomObject]@{
            SiteUrl      = $site.Url
            ErrorMessage = $_.Exception.Message
            ErrorDetails = $_.Exception.StackTrace
        }
    }
}

# === Export results to CSV ===
$alertResults | Export-Csv -Path $outputCsvPath -NoTypeInformation -Encoding UTF8
Write-Host "`nReport exported to: $outputCsvPath" -ForegroundColor Green

# === Export errors if any ===
if ($errorLog.Count -gt 0) {
    $errorLog | Export-Csv -Path $errorLogPath -NoTypeInformation -Encoding UTF8
    Write-Host "Error log exported to: $errorLogPath" -ForegroundColor Yellow
} else {
    Write-Host "No errors encountered." -ForegroundColor Green
}

# === Disconnect session ===
Disconnect-PnPOnline
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
# .\Export-SPOAlerts.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -ReportPath ".\reports\spo-alerts.csv" -ErrorPath ".\reports\spo-alerts-errors.csv" -WhatIf
[CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'Medium')]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint Online Admin Center URL")]
    [ValidatePattern('^https://')]
    [string]$AdminUrl,

    [Parameter(Mandatory = $true, HelpMessage = "CSV path for the alert report")]
    [ValidateNotNullOrEmpty()]
    [string]$ReportPath,

    [Parameter(Mandatory = $true, HelpMessage = "CSV path for operation errors")]
    [ValidateNotNullOrEmpty()]
    [string]$ErrorPath
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 session."
    m365 login --ensure

    $Script:AlertReport = [System.Collections.Generic.List[pscustomobject]]::new()
    $Script:AlertErrors = [System.Collections.Generic.List[pscustomobject]]::new()
}

process {
    Write-Verbose "Retrieving site collection inventory."
    $sitesJson = m365 spo site list --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to list sites. CLI output: $sitesJson"
    }
    $sites = $sitesJson | ConvertFrom-Json

    foreach ($site in $sites) {
        Write-Verbose "Processing site $($site.Url)"

        $alertJson = m365 spo web alert list --webUrl $site.Url --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            $Script:AlertErrors.Add([pscustomobject]@{
                SiteUrl = $site.Url
                Error   = $alertJson
            })
            continue
        }

        if ([string]::IsNullOrWhiteSpace(($alertJson | Out-String).Trim())) {
            continue
        }

        try {
            $alertObjects = $alertJson | ConvertFrom-Json
        }
        catch {
            $Script:AlertErrors.Add([pscustomobject]@{
                SiteUrl = $site.Url
                Error   = "Failed to parse alert response. $_"
            })
            continue
        }

        if (-not $alertObjects) {
            continue
        }

        foreach ($alert in @($alertObjects)) {
            $user = $alert.User
            $list = $alert.List
            $item = $alert.Item

            $Script:AlertReport.Add([pscustomobject]@{
                SiteUrl           = $site.Url
                Title             = $alert.Title
                UserName          = if ($user) { $user.Title } else { $null }
                UserPrincipalName = if ($user) { $user.UserPrincipalName } else { $null }
                UserEmail         = if ($user) { $user.Email } else { $null }
                AlertType         = $alert.AlertType
                EventType         = $alert.EventType
                Frequency         = $alert.AlertFrequency
                Status            = $alert.Status
                DeliveryChannels  = $alert.DeliveryChannels
                ListTitle         = if ($list) { $list.Title } else { $null }
                ListUrl           = if ($list -and $list.RootFolder) { $list.RootFolder.ServerRelativeUrl } else { $null }
                ItemUrl           = if ($item) { $item.FileRef } else { $null }
                LastModified      = $alert.LastModified
            })
        }
    }
}

end {
    if ($Script:AlertReport.Count -gt 0) {
        if ($PSCmdlet.ShouldProcess($ReportPath, 'Export alert report')) {
            $reportDir = Split-Path -Path $ReportPath -Parent
            if ($reportDir -and -not (Test-Path $reportDir)) {
                New-Item -ItemType Directory -Path $reportDir -Force | Out-Null
            }
            $Script:AlertReport | Sort-Object SiteUrl, Title | Export-Csv -Path $ReportPath -NoTypeInformation
            Write-Host "Alert report exported to '$ReportPath'" -ForegroundColor Green
        }
    }
    else {
        Write-Host "No classic SharePoint alerts were discovered." -ForegroundColor Yellow
    }

    if ($Script:AlertErrors.Count -gt 0) {
        if ($PSCmdlet.ShouldProcess($ErrorPath, 'Export alert errors')) {
            $errorDir = Split-Path -Path $ErrorPath -Parent
            if ($errorDir -and -not (Test-Path $errorDir)) {
                New-Item -ItemType Directory -Path $errorDir -Force | Out-Null
            }
            $Script:AlertErrors | Export-Csv -Path $ErrorPath -NoTypeInformation
            Write-Host "Errors encountered; see '$ErrorPath'" -ForegroundColor Yellow
        }
    }
    else {
        Write-Host "No errors encountered." -ForegroundColor Green
    }
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author |
|-----------|
| [Tanel Vahk](https://www.linkedin.com/in/tvahk/) |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-sharepoint-alerts-audit" aria-hidden="true" />

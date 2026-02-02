# Find all Remote Event Receivers in a SharePoint Online site

## Summary

Remote Event Receivers (RERs) are a way to extend the functionality of SharePoint Online by allowing developers to execute custom code in response to specific events that occur within a SharePoint site. However, starting April 2. 2026, Microsoft is deprecating RERs in favor of more modern approaches like Power Automate and webhooks. This script helps administrators identify and manage existing RERs before the deprecation date.

This script will enumerate all Remote Event Receivers in all SharePoint Online site collections, filter out internal Microsoft event receivers, and export the results to CSV with details like site URL, list title, receiver name, URL, event type, and ID.


# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint tenant URL (e.g., https://contoso.sharepoint.com)")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$TenantUrl
)

begin {
    Write-Host "Starting Remote Event Receiver audit..." -ForegroundColor Cyan

    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to Microsoft 365"
    }

    $script:ReceiverCollection = @()
    $script:Summary = @{
        TotalSites = 0
        ListsChecked = 0
        RemoteReceiversFound = 0
        Failures = 0
    }

    $script:TranscriptPath = "RemoteEventReceivers-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
    Start-Transcript -Path $script:TranscriptPath | Out-Null
    Write-Verbose "Transcript logging started: $script:TranscriptPath"
}

process {
    try {
        Write-Host "Retrieving all site collections..." -ForegroundColor Cyan
        $sitesJson = m365 spo site list --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve site collections"
        }

        $sites = $sitesJson | ConvertFrom-Json
        $script:Summary.TotalSites = $sites.Count
        Write-Host "Found $($sites.Count) site collections" -ForegroundColor Cyan

        foreach ($site in $sites) {
            Write-Host "Processing site: $($site.Url)" -ForegroundColor Yellow

            try {
                $listsJson = m365 spo list list --webUrl $site.Url --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve lists for site: $($site.Url)"
                    $script:Summary.Failures++
                    continue
                }

                $lists = $listsJson | ConvertFrom-Json

                foreach ($list in $lists) {
                    $script:Summary.ListsChecked++
                    Write-Verbose "  Checking list: $($list.Title)"

                    try {
                        $receiversJson = m365 spo eventreceiver list --webUrl $site.Url --listTitle $list.Title --output json 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Verbose "    No event receivers or error for list: $($list.Title)"
                            continue
                        }

                        $receivers = $receiversJson | ConvertFrom-Json

                        foreach ($receiver in $receivers) {
                            if ($null -ne $receiver.ReceiverUrl -and $receiver.ReceiverUrl -ne "") {
                                $urlObject = [System.Uri]::new($receiver.ReceiverUrl)
                                if (-not $urlObject.Host.EndsWith("svc.ms")) {
                                    Write-Host "    Found remote event receiver: $($receiver.ReceiverName) at $($receiver.ReceiverUrl)" -ForegroundColor Green
                                    
                                    $script:ReceiverCollection += [PSCustomObject]@{
                                        SiteUrl = $site.Url
                                        ListTitle = $list.Title
                                        ReceiverName = $receiver.ReceiverName
                                        ReceiverUrl = $receiver.ReceiverUrl
                                        EventType = $receiver.EventType
                                        ReceiverId = $receiver.ReceiverId
                                    }
                                    $script:Summary.RemoteReceiversFound++
                                }
                            }
                        }
                    }
                    catch {
                        Write-Warning "    Error processing receivers for list '$($list.Title)': $($_.Exception.Message)"
                        continue
                    }
                }
            }
            catch {
                Write-Warning "Error processing site '$($site.Url)': $($_.Exception.Message)"
                $script:Summary.Failures++
                continue
            }
        }
    }
    catch {
        Write-Error "Critical error: $($_.Exception.Message)"
        throw
    }
}

end {
    $csvPath = "RemoteEventReceivers-$(Get-Date -Format 'yyyyMMdd-HHmmss').csv"
    
    if ($script:ReceiverCollection.Count -gt 0) {
        $script:ReceiverCollection | Export-Csv -Path $csvPath -NoTypeInformation
        Write-Host "`nReport exported to: " -NoNewline -ForegroundColor Gray
        Write-Host $csvPath -ForegroundColor Cyan
    }
    else {
        Write-Host "`nNo remote event receivers found" -ForegroundColor Green
    }

    Write-Host "`n=== Summary ===" -ForegroundColor Cyan
    Write-Host "Sites processed:          " -NoNewline -ForegroundColor Gray
    Write-Host $script:Summary.TotalSites -ForegroundColor White
    Write-Host "Lists checked:            " -NoNewline -ForegroundColor Gray
    Write-Host $script:Summary.ListsChecked -ForegroundColor White
    Write-Host "Remote receivers found:   " -NoNewline -ForegroundColor Gray
    if ($script:Summary.RemoteReceiversFound -gt 0) {
        Write-Host $script:Summary.RemoteReceiversFound -ForegroundColor Yellow
    }
    else {
        Write-Host $script:Summary.RemoteReceiversFound -ForegroundColor Green
    }
    Write-Host "Failures:                 " -NoNewline -ForegroundColor Gray
    if ($script:Summary.Failures -gt 0) {
        Write-Host $script:Summary.Failures -ForegroundColor Red
    }
    else {
        Write-Host $script:Summary.Failures -ForegroundColor Green
    }
    Write-Host "`nTranscript:               " -NoNewline -ForegroundColor Gray
    Write-Host $script:TranscriptPath -ForegroundColor Cyan

    Stop-Transcript | Out-Null
}

# Example 1: Basic usage - scan all sites for remote event receivers
# .\Find-RemoteEventReceivers.ps1 -TenantUrl "https://contoso.sharepoint.com"

# Example 2: With verbose output to see all lists being checked
# .\Find-RemoteEventReceivers.ps1 -TenantUrl "https://contoso.sharepoint.com" -Verbose

# Example 3: Scan specific government cloud tenant
# .\Find-RemoteEventReceivers.ps1 -TenantUrl "https://contoso.sharepoint.us"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]


# [PnP PowerShell](#tab/pnpps)

```powershell

$TenantUrl = "https://<tenant>.sharepoint.com/";

# I recommend using app-only authentication for this script
Connect-PnPOnline -Url $TenantUrl -ClientId "<client-id>" -Tenant "<tenant>.onmicrosoft.com" -CertificatePath "<path-to-certificate>" -CertificatePassword (ConvertTo-SecureString "<certificate-password>" -AsPlainText -Force);

$Sites = Get-PnPTenantSite;

$ReceiverInfo = @();


foreach ($Site in $Sites) {
    Connect-PnPOnline -Url $Site.Url;
    Write-Host "Processing site: $($Site.Url)" -ForegroundColor Cyan;
    $Lists = Get-PnPList;
    foreach ($List in $Lists) {
        Write-Host "`t>Processing list: $($List.Title)" -ForegroundColor Yellow;
        $EventReceivers = Get-PnPEventReceiver -List $List;

        foreach ($Receiver in $EventReceivers) {
            if ($null -ne $Receiver.ReceiverUrl) {
                $UrlObject = [System.Uri]::new($Receiver.ReceiverUrl);
                if (-not $UrlObject.IsBaseOf("svc.ms")) {
                    Write-Host "`t`t>Found remote event receiver: $($Receiver.ReceiverName) at $($Receiver.ReceiverUrl)" -ForegroundColor Green;
                    $ReceiverInfo += [PSCustomObject]@{
                        SiteUrl      = $Site.Url;
                        ListTitle    = $List.Title;
                        ReceiverName = $Receiver.ReceiverName;
                        ReceiverUrl  = $Receiver.ReceiverUrl;
                    };
                }
            }
        }
    }
}

$ReceiverInfo | Export-Csv -Path "RemoteEventReceivers.csv" -NoTypeInformation; 

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| Dan Toft |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-remote-event-receivers" aria-hidden="true" />

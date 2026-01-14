

# Gets usage from a particular user(s) or site(s) from the Unified Audit Log

## Summary

Say we have a user who has written a lot of flows and PowerBI reports and is complaining that she is getting throttled in SharePoint quite often.

We need to see all the calls being made by that user and/or to particular sites to attempt to narrow down the issues.

This script will scan the ULS Logs for the last week looking for all access by a user an or to a site and create an excel file summarizing the activity.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(HelpMessage = "Time interval in minutes for each audit log query batch (default: 15)")]
    [int]$IntervalMinutes = 15,
    
    [Parameter(Mandatory, HelpMessage = "How many minutes to look back (max 10080 for 7 days)")]
    [ValidateRange(1, 10080)]
    [int]$LookbackMinutes,
    
    [Parameter(HelpMessage = "Filter by specific user UPNs (e.g., 'user1@contoso.com','user2@contoso.com')")]
    [string[]]$UserIds,
    
    [Parameter(HelpMessage = "Filter by specific site URLs (e.g., 'https://contoso.sharepoint.com/sites/Site1')")]
    [string[]]$SiteUrls,
    
    [Parameter(HelpMessage = "Output directory path (default: current location)")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' manually."
    }

    if ($PSBoundParameters.ContainsKey('OutputPath') -and -not (Test-Path -Path $OutputPath -PathType Container)) {
        throw "Output path '$OutputPath' does not exist. Please create it first."
    }

    $script:OutputArray = @()
    $script:Summary = @{
        IntervalsProcessed = 0
        TotalLogsRetrieved = 0
        FilteredLogs = 0
        Failures = 0
    }

    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "AuditLogRetrieval_$timestamp.log"
    Start-Transcript -Path $transcriptPath | Out-Null

    Write-Host "Starting audit log retrieval..." -ForegroundColor Cyan
    Write-Host "Lookback period: $LookbackMinutes minutes ($(([math]::Round($LookbackMinutes / 1440, 2))) days)" -ForegroundColor Cyan
    if ($UserIds) {
        Write-Host "Filtering by users: $($UserIds -join ', ')" -ForegroundColor Cyan
    }
    if ($SiteUrls) {
        Write-Host "Filtering by sites: $($SiteUrls -join ', ')" -ForegroundColor Cyan
    }
}

process {
    $totalIntervals = [math]::Ceiling($LookbackMinutes / $IntervalMinutes)
    $currentInterval = 0

    for ($i = $LookbackMinutes; $i -gt 0; $i -= $IntervalMinutes) {
        $currentInterval++
        $minutesBack = $i
        $minutesForward = [math]::Max(0, $i - $IntervalMinutes)
        
        $startTime = (Get-Date).AddMinutes(-$minutesBack).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss.fffZ")
        $endTime = (Get-Date).AddMinutes(-$minutesForward).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss.fffZ")

        Write-Verbose "Processing interval $currentInterval/$totalIntervals (Start: $startTime, End: $endTime)"

        try {
            $auditLogs = m365 purview auditlog list --contentType SharePoint --startTime $startTime --endTime $endTime --output json 2>&1
            
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve audit logs for interval $currentInterval. Error: $auditLogs"
                $script:Summary.Failures++
                
                $script:OutputArray += [PSCustomObject]@{
                    IntervalNumber = $currentInterval
                    StartTime = $startTime
                    EndTime = $endTime
                    TotalLogs = 0
                    FilteredLogsCount = 0
                    UniqueUsers = 0
                    UniqueSites = 0
                    TopOperations = ""
                    Notes = "Failed to retrieve logs: $auditLogs"
                }
                continue
            }

            $logs = @($auditLogs | ConvertFrom-Json)
            $script:Summary.TotalLogsRetrieved += $logs.Count

            $filteredLogs = $logs
            if ($UserIds -or $SiteUrls) {
                $filteredLogs = $logs | Where-Object {
                    $matchUser = $true
                    $matchSite = $true
                    
                    if ($UserIds) {
                        $matchUser = $UserIds -contains $_.UserId
                    }
                    if ($SiteUrls) {
                        $currentLog = $_
                        $matchSite = $false
                        foreach ($siteUrl in $SiteUrls) {
                            if ($currentLog.ObjectId -like "*$siteUrl*" -or $currentLog.SiteUrl -like "*$siteUrl*") {
                                $matchSite = $true
                                break
                            }
                        }
                    }
                    
                    return ($matchUser -and $matchSite)
                }
            }

            $script:Summary.FilteredLogs += $filteredLogs.Count

            $uniqueUsers = ($filteredLogs | Select-Object -ExpandProperty UserId -Unique).Count
            $uniqueSites = ($filteredLogs | Select-Object -ExpandProperty SiteUrl -Unique | Where-Object { $_ }).Count
            $topOperations = ($filteredLogs | Group-Object -Property Operation | Sort-Object -Property Count -Descending | Select-Object -First 5 -ExpandProperty Name) -join '|'

            $script:OutputArray += [PSCustomObject]@{
                IntervalNumber = $currentInterval
                StartTime = $startTime
                EndTime = $endTime
                TotalLogs = $logs.Count
                FilteredLogsCount = $filteredLogs.Count
                UniqueUsers = $uniqueUsers
                UniqueSites = $uniqueSites
                TopOperations = $topOperations
                Notes = ""
            }

            $script:Summary.IntervalsProcessed++

            if ($currentInterval % 10 -eq 0) {
                Write-Host "Progress: Processed $currentInterval/$totalIntervals intervals..." -ForegroundColor Gray
            }

        } catch {
            Write-Warning "Exception during interval $currentInterval processing: $($_.Exception.Message)"
            $script:Summary.Failures++
            
            $script:OutputArray += [PSCustomObject]@{
                IntervalNumber = $currentInterval
                StartTime = $startTime
                EndTime = $endTime
                TotalLogs = 0
                FilteredLogsCount = 0
                UniqueUsers = 0
                UniqueSites = 0
                TopOperations = ""
                Notes = "Exception: $($_.Exception.Message)"
            }
            continue
        }
    }
}

end {
    $csvPath = Join-Path $OutputPath "AuditLogSummary_$timestamp.csv"
    $script:OutputArray | Export-Csv -Path $csvPath -NoTypeInformation

    Write-Host "`n========================================" -ForegroundColor Green
    Write-Host "Audit Log Retrieval Summary" -ForegroundColor Green
    Write-Host "========================================" -ForegroundColor Green
    Write-Host "Intervals Processed: $($script:Summary.IntervalsProcessed)/$totalIntervals" -ForegroundColor White
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Green
    }
    
    Write-Host "Total Logs Retrieved: $($script:Summary.TotalLogsRetrieved)" -ForegroundColor White
    Write-Host "Filtered Logs: $($script:Summary.FilteredLogs)" -ForegroundColor White
    Write-Host "`nCSV Report: $csvPath" -ForegroundColor Cyan
    Write-Host "Transcript Log: $transcriptPath" -ForegroundColor Cyan
    Write-Host "========================================`n" -ForegroundColor Green

    Stop-Transcript | Out-Null
}

# Basic usage - retrieve last 1440 minutes (1 day) of audit logs
# .\Get-AuditLogUsage.ps1 -LookbackMinutes 1440

# Filter by specific users
# .\Get-AuditLogUsage.ps1 -LookbackMinutes 1440 -UserIds "user1@contoso.com","user2@contoso.com"

# Filter by specific sites
# .\Get-AuditLogUsage.ps1 -LookbackMinutes 1440 -SiteUrls "https://contoso.sharepoint.com/sites/Site1","https://contoso.sharepoint.com/sites/Site2"

# Filter by both users and sites with custom interval and output path
# .\Get-AuditLogUsage.ps1 -LookbackMinutes 10080 -IntervalMinutes 30 -UserIds "user@contoso.com" -SiteUrls "https://contoso.sharepoint.com/sites/Site1" -OutputPath "C:\Reports" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell
Connect-PnPOnline -Url "HTTPS://tenant-ADMIN.sharepoint.COM" -Interactive
$intervalminutes = 15 
$now = Get-Date
$outputArray = @()
for ($i = 60; $i -le 11000 ; $i = $i + $intervalminutes) {
    # 1 hour ago to a day ago
    $starttime = $now.AddMinutes(-$i - $intervalminutes)
    $endtime = $now.AddMinutes(-$i)
    $results = Get-PnPUnifiedAuditLog -ContentType "SharePoint" -StartTime $starttime -EndTime $endtime
    $OperationalExcellenceHub = $results | Where { $_.SiteUrl -eq "https://tenant.sharepoint.com/sites/OperationalExcellenceHub/" }
    $OperationalExcellence = $results | Where { $_.SiteUrl -eq "https://tenant.sharepoint.com/sites/OperationalExcellence/" }
    $user= $results | Where { $_.UserId -eq "some.user@domain.com" }
    Write-Host  "$i FROM $starttime TO $endtime  OperationalExcellenceHub:$($OperationalExcellenceHub.Count) OperationalExcellence:$($OperationalExcellence.Count) RobS:$($Sarracini.Count) TOTAL:$($results.Count)"
    $outputObject = [PSCustomObject]@{
        Count                    = $i
        StartTime                = $starttime
        EndTime                  = $endtime
        OperationalExcellenceHub = $OperationalExcellenceHub.Count
        OperationalExcellence    = $OperationalExcellence.Count
        Sarracini                = $Sarracini.Count
        Total                    = $results.Count
    }
    $outputArray += $outputObject
    
}

$outputArray | Export-Csv "c:\Temp\IOCounts.csv" -NoTypeInformation

    # End

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| Russell Gove |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-usage-from-audit-logs" aria-hidden="true" />

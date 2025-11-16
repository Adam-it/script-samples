

# Export Inactive Sites Based On Days To CSV

## Summary
This sample demonstrates the process of exporting sites that have been inactive for a specified number of days (in this script, we've set it to the last 30 days) to a CSV file. The exported report encompasses all columns, such as URL, Title, Template, and more.

![Example Screenshot](assets/example.png)

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
function Export-InactiveSpoSites {
    [CmdletBinding()]
    param(
        [Parameter(HelpMessage = "Number of days of inactivity to include (default: 30)")]
        [ValidateRange(1, 3650)]
        [int]
        $DaysInactive = 30,

        [Parameter(HelpMessage = "Directory where the CSV report will be stored")]
        [ValidateNotNullOrEmpty()]
        [string]
        $ReportDirectory = "./reports",

        [Parameter(HelpMessage = "Include OneDrive for Business sites in the inventory")]
        [switch]
        $IncludeOneDriveSites,

        [Parameter(HelpMessage = "URL patterns to exclude from processing")]
        [string[]]
        $ExcludeUrlPatterns = @('*-my.sharepoint.com*', '*/portals/*')
    )

    begin {
        Write-Host "Ensuring Microsoft 365 CLI session..." -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to ensure Microsoft 365 CLI login. CLI output: $loginOutput"
        }

        $resolvedDirectory = Resolve-Path -Path $ReportDirectory -ErrorAction SilentlyContinue
        if (-not $resolvedDirectory) {
            Write-Verbose "Creating report directory '$ReportDirectory'."
            $null = New-Item -ItemType Directory -Path $ReportDirectory -Force
            $resolvedDirectory = Resolve-Path -Path $ReportDirectory
        }

        $script:ReportPath = Join-Path -Path $resolvedDirectory.Path -ChildPath ("inactive-sites-{0}.csv" -f (Get-Date -Format 'yyyyMMdd-HHmmss'))
        $script:ThresholdDate = (Get-Date).AddDays(-$DaysInactive)
        $script:Summary = [ordered]@{
            TotalSitesReturned = 0
            EvaluatedSites     = 0
            ExcludedSites      = 0
            InactiveSites      = 0
            Failures           = 0
        }
        $script:Results = New-Object System.Collections.Generic.List[object]

        function Convert-SpoDateString {
            param(
                [Parameter()] [string] $Value
            )

            if ([string]::IsNullOrWhiteSpace($Value)) {
                return $null
            }

            $trimmed = $Value.Trim('/')

            if ($trimmed -match 'Date\((\d+),(\d+),(\d+),(\d+),(\d+),(\d+),(\d+)\)') {
                try {
                    return [datetime]::SpecifyKind([datetime]::new([int]$Matches[1], [int]$Matches[2], [int]$Matches[3], [int]$Matches[4], [int]$Matches[5], [int]$Matches[6], [int]$Matches[7]), [datetimekind]::Utc)
                }
                catch {
                    Write-Verbose "Failed to parse detailed SPO date '$Value': $($_.Exception.Message)"
                    return $null
                }
            }

            if ($trimmed -match 'Date\(([-]?\d+)\)') {
                try {
                    return [DateTimeOffset]::FromUnixTimeMilliseconds([int64]$Matches[1]).UtcDateTime
                }
                catch {
                    Write-Verbose "Failed to parse epoch SPO date '$Value': $($_.Exception.Message)"
                    return $null
                }
            }

            try {
                return [datetime]::Parse($Value, [System.Globalization.CultureInfo]::InvariantCulture)
            }
            catch {
                Write-Verbose "Unrecognised date format '$Value'."
                return $null
            }
        }

        Write-Host "Looking for SharePoint sites inactive for at least $DaysInactive day(s)..." -ForegroundColor Yellow
    }

    process {
        $siteArgs = @(
            'spo', 'site', 'list',
            '--output', 'json',
            '--query', '[].{Title:Title,Url:Url,Template:Template,LastContentModifiedDate:LastContentModifiedDate,StorageUsage:StorageUsage}'
        )

        if ($IncludeOneDriveSites) {
            $siteArgs += '--withOneDriveSites'
        }

        $siteOutput = m365 @siteArgs 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve SharePoint sites. CLI output: $siteOutput"
        }

        try {
            $sites = $siteOutput | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse CLI response as JSON. $($_.Exception.Message)"
        }

        if (-not $sites) {
            Write-Warning "Microsoft 365 CLI returned no sites."
            return
        }

        $script:Summary.TotalSitesReturned = $sites.Count
        Write-Host "Evaluating $($sites.Count) site(s)..." -ForegroundColor Cyan

        $now = Get-Date

        foreach ($site in $sites) {
            if ($ExcludeUrlPatterns -and ($ExcludeUrlPatterns | Where-Object { $site.Url -like $_ })) {
                $script:Summary.ExcludedSites++
                continue
            }

            $script:Summary.EvaluatedSites++
            $lastModified = Convert-SpoDateString -Value $site.LastContentModifiedDate
            $isInactive = $true

            if ($lastModified) {
                $isInactive = $lastModified -le $script:ThresholdDate
            }

            if (-not $isInactive) {
                continue
            }

            $script:Summary.InactiveSites++
            $daysSince = if ($lastModified) { [math]::Floor(($now - $lastModified).TotalDays) } else { $null }

            $script:Results.Add([pscustomobject]@{
                Title                      = $site.Title
                Url                        = $site.Url
                Template                   = $site.Template
                LastContentModifiedDateUtc = if ($lastModified) { $lastModified.ToString('u') } else { 'Unknown' }
                DaysSinceLastActivity      = $daysSince
                StorageUsageMB             = $site.StorageUsage
            })
        }
    }

    end {
        if ($script:Results.Count -gt 0) {
            try {
                $script:Results | Export-Csv -Path $script:ReportPath -NoTypeInformation -Encoding UTF8
                Write-Host "Inactive site report saved to $($script:ReportPath)." -ForegroundColor Green
            }
            catch {
                $script:Summary.Failures++
                Write-Error "Failed to write CSV report. $($_.Exception.Message)"
            }
        }
        else {
            Write-Host "No sites were inactive for at least $DaysInactive day(s)." -ForegroundColor Green
        }

        Write-Host "----- Summary -----" -ForegroundColor Cyan
        Write-Host "Sites returned       : $($script:Summary.TotalSitesReturned)"
        Write-Host "Sites evaluated      : $($script:Summary.EvaluatedSites)"
        Write-Host "Sites excluded       : $($script:Summary.ExcludedSites)"
        Write-Host "Inactive sites found : $($script:Summary.InactiveSites)"
        if ($script:Results.Count -gt 0 -and (Test-Path -Path $script:ReportPath)) {
            Write-Host "Report path          : $($script:ReportPath)" -ForegroundColor Cyan
        }
        if ($script:Summary.Failures -gt 0) {
            Write-Warning "Number of failures: $($script:Summary.Failures)"
        }
    }
}

Export-InactiveSpoSites -DaysInactive 45 -ReportDirectory "./reports"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

$adminSiteURL = "https://domain-admin.sharepoint.com/"
$username = "username@domain.onmicrosoft.com"
$password = "********"
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd
$dateTime = "_{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$basePath = "D:\Contributions\Scripts\Logs\"
$csvPath = $basePath + "\InActiveSites_PnP" + $dateTime + ".csv"
$global:inActiveSites = @()
$daysInActive = 30

Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $creds)     
    Write-Host "Connecting to Site '$($adminSiteURL)'..." -ForegroundColor Yellow   
    Connect-PnPOnline -Url $adminSiteURL -Credential $creds
    Write-Host "Connection Successful!" -ForegroundColor Green 
}

Function GetInactiveSites {    
    try {
        Write-Host "Getting inactive sites..." -ForegroundColor Yellow 
        $siteCollections = Get-PnPTenantSite | Where-Object {$_.Url -notlike "-my.sharepoint.com" -and $_.Url -notlike "/portals/"}
         
        #calculate the Date
        $date = (Get-Date).AddDays(-$daysInActive).ToString("MM/dd/yyyy")
 
        #Get inactive sites based on modified date
        $global:inActiveSites = $siteCollections | Where {$_.LastContentModifiedDate -le $date} | Select *         
        Write-Host "Getting inactive sites successfully!"  -ForegroundColor Green 
    }
    catch {
        Write-Host "Error in getting inactive sites:" $_.Exception.Message -ForegroundColor Red                 
    }
    Write-Host "Exporting to CSV..."  -ForegroundColor Yellow 
    $global:inActiveSites | Export-Csv $csvPath -NoTypeInformation -Append
    Write-Host "Exported to CSV successfully!"  -ForegroundColor Green	
}

Function StartProcessing {
    Login($creds);
    GetInactiveSites
}

StartProcessing
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [SPO Management Shell](#tab/spoms-ps)

```powershell
$adminSiteURL = "https://domain-admin.sharepoint.com/"
$username = "username@domain.onmicrosoft.com"
$password = "********"
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd
$dateTime = "_{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$basePath = "D:\Contributions\Scripts\Logs\"
$csvPath = $basePath + "\InActiveSites_SPO" + $dateTime + ".csv"
$global:inActiveSites = @()
$daysInActive = 30

Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $creds)     
    Write-Host "Connecting to Site '$($adminSiteURL)'..." -ForegroundColor Yellow   
    Connect-SPOService -Url $adminSiteURL -Credential $creds
    Write-Host "Connection Successful!" -ForegroundColor Green 
}

Function GetInactiveSites {    
    try {
        Write-Host "Getting inactive sites..." -ForegroundColor Yellow 
        $siteCollections = Get-SPOSite -Filter { Url -notlike "-my.sharepoint.com" -and Url -notlike "/portals/" }
         
        #Calculate the Date
        $date = (Get-Date).AddDays(-$daysInActive).ToString("MM/dd/yyyy")
 
        #Get All Site collections where the content modified
        $global:inActiveSites = $siteCollections | Where {$_.LastContentModifiedDate -le $date} | Select *         
        Write-Host "Getting inactive sites successfully!"  -ForegroundColor Green 
    }
    catch {
        Write-Host "Error in getting inactive sites:" $_.Exception.Message -ForegroundColor Red                 
    }
    Write-Host "Exporting to CSV..."  -ForegroundColor Yellow 
    $global:inActiveSites | Export-Csv $csvPath -NoTypeInformation -Append
    Write-Host "Exported to CSV successfully!"  -ForegroundColor Green	
}

Function StartProcessing {
    Login($creds);
    GetInactiveSites
}

StartProcessing
```
[!INCLUDE [More about SPO Management Shell](../../docfx/includes/MORE-SPOMS.md)]
***
## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati (https://github.com/chandaniprajapati) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/export-inactive-sites-based-on-days-to-csv" aria-hidden="true" />

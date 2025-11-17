

# Export OneDrive Sites

## Summary
This sample shows how to export all onedrive sites to CSV. This report includes all columns. for eg. Url, Owner, Storage and etc.

![Example Screenshot](assets/example.png)

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
function Export-OneDriveSitesToCsv {
    [CmdletBinding()]
    param(
        [Parameter(HelpMessage = "Path where the CSV report should be stored")]
        [ValidateNotNullOrEmpty()]
        [string]
        $ReportDirectory = "./reports",

        [Parameter(HelpMessage = "Additional columns to export from SPO site metadata")]
        [string[]]
        $AdditionalFields = @(),

        [Parameter(HelpMessage = "Skip personal sites whose state is NotProvisioned or Recycled")]
        [switch]
        $SkipInactiveSites
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

        $script:ReportPath = Join-Path -Path $resolvedDirectory.Path -ChildPath ("onedrive-sites-{0}.csv" -f (Get-Date -Format 'yyyyMMdd-HHmmss'))
        $script:Summary = [ordered]@{
            SitesReturned = 0
            SitesExported = 0
            SitesSkipped  = 0
            Failures      = 0
        }
        $script:Results = New-Object System.Collections.Generic.List[object]

        # Default list of fields to export. Additional fields will be merged in if provided.
        $script:BaseFields = @(
            'Title',
            'Url',
            'Owner',
            'StorageUsage',
            'LastContentModifiedDate',
            'Status',
            'LockIssue'
        )

        if ($AdditionalFields.Length -gt 0) {
            $script:ExportFields = ($script:BaseFields + $AdditionalFields) | Select-Object -Unique
        }
        else {
            $script:ExportFields = $script:BaseFields
        }

        Write-Verbose "Exporting fields: $($script:ExportFields -join ', ')"
    }

    process {
        $queryFields = $script:ExportFields + @('Template')
        $queryProjection = $queryFields | ForEach-Object { "$_:$_" }
        $query = "[].{0}" -f (($queryProjection -join ',').TrimEnd(','))

        $siteOutput = m365 spo site list --withOneDriveSites --output json --query $query 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve OneDrive sites. CLI output: $siteOutput"
        }

        try {
            $sites = $siteOutput | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse CLI response as JSON. $($_.Exception.Message)"
        }

        if (-not $sites) {
            Write-Warning "Microsoft 365 CLI returned no OneDrive sites."
            return
        }

        $script:Summary.SitesReturned = $sites.Count
        Write-Host "Processing $($sites.Count) OneDrive site(s)..." -ForegroundColor Cyan

        foreach ($site in $sites) {
            if ($SkipInactiveSites -and ($site.Status -in @('NotProvisioned', 'Recycled'))) {
                $script:Summary.SitesSkipped++
                continue
            }

            try {
                $record = [ordered]@{
                    Url    = $site.Url
                    Owner  = $site.Owner
                    Title  = $site.Title
                    Status = $site.Status
                }

                if ($site.StorageUsage) {
                    $record.StorageUsageMB = [math]::Round([double]$site.StorageUsage, 2)
                }
                else {
                    $record.StorageUsageMB = $null
                }

                if ($site.LastContentModifiedDate) {
                    $record.LastContentModifiedDateUtc = $site.LastContentModifiedDate
                }

                if ($site.LockIssue) {
                    $record.LockIssue = $site.LockIssue
                }

                foreach ($field in $AdditionalFields) {
                    if (-not $record.Contains($field)) {
                        $record[$field] = if ($site.PSObject.Properties[$field]) { $site.$field } else { $null }
                    }
                }

                $script:Results.Add([pscustomobject]$record)
                $script:Summary.SitesExported++
            }
            catch {
                $script:Summary.Failures++
                Write-Warning "Failed to process site '$($site.Url)': $($_.Exception.Message)"
            }
        }
    }

    end {
        if ($script:Results.Count -gt 0) {
            try {
                $script:Results | Export-Csv -Path $script:ReportPath -NoTypeInformation -Encoding UTF8
                Write-Host "OneDrive inventory exported to $($script:ReportPath)." -ForegroundColor Green
            }
            catch {
                $script:Summary.Failures++
                Write-Error "Failed to write CSV report. $($_.Exception.Message)"
            }
        }
        else {
            Write-Host "No OneDrive sites met the export criteria." -ForegroundColor Green
        }

        Write-Host "----- Summary -----" -ForegroundColor Cyan
        Write-Host "Sites returned  : $($script:Summary.SitesReturned)"
        Write-Host "Sites exported  : $($script:Summary.SitesExported)"
        Write-Host "Sites skipped   : $($script:Summary.SitesSkipped)"
        if ($script:Results.Count -gt 0 -and (Test-Path -Path $script:ReportPath)) {
            Write-Host "Report path     : $($script:ReportPath)" -ForegroundColor Cyan
        }
        if ($script:Summary.Failures -gt 0) {
            Write-Warning "Number of failures: $($script:Summary.Failures)"
        }
    }
}

Export-OneDriveSitesToCsv -SkipInactiveSites -AdditionalFields Template,StorageWarningLevel
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
$basePath = "E:\Contribution\PnP-Scripts\Logs\"
$csvPath = $basePath + "\OneDriveSites_PnP" + $dateTime + ".csv"
$global:onedriveSitesCollection = @()

Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $creds)     
    Write-Host "Connecting to Site '$($adminSiteURL)'" -f Yellow   
    Connect-PnPOnline -Url $adminSiteURL -Credential $creds
    Write-Host "Connection Successful" -f Green 
}

Function GetOnedriveSitesDeails {    
    try {
        Write-Host "Getting onedrive sites..."  -ForegroundColor Yellow 
        $global:onedriveSitesCollection = Get-PnPTenantSite -IncludeOneDriveSites -Filter "Url -like '-my.sharepoint.com/personal/'" | select *          
    }
    catch {
        Write-Host "Error in getting onedrive sites:" $_.Exception.Message -ForegroundColor Red                 
    }
    Write-Host "Exporting to CSV..."  -ForegroundColor Yellow 
    $global:onedriveSitesCollection | Export-Csv $csvPath -NoTypeInformation -Append
    Write-Host "Exported to CSV successfully!"  -ForegroundColor Green	

}

Function StartProcessing {
    Login($creds);
    GetOnedriveSitesDeails
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
$basePath = "E:\Contribution\PnP-Scripts\Logs\"
$csvPath = $basePath + "\OneDriveSites_SPO" + $dateTime + ".csv"
$global:onedriveSitesCollection = @()

Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $creds)     
    Write-Host "Connecting to Site '$($adminSiteURL)'" -f Yellow   
    Connect-SPOService -Url $adminSiteURL -Credential $creds
    Write-Host "Connection Successful" -f Green 
}

Function GetOnedriveSitesDeails {    
    try {
        Write-Host "Getting onedrive sites..."  -ForegroundColor Yellow 
        $global:onedriveSitesCollection = Get-SPOSite -Template "SPSPERS" -limit ALL -includepersonalsite $True | select *        
    }
    catch {
        Write-Host "Error in getting onedrive sites:" $_.Exception.Message -ForegroundColor Red                 
    }
    Write-Host "Exporting to CSV..."  -ForegroundColor Yellow 
    $global:onedriveSitesCollection | Export-Csv $csvPath -NoTypeInformation -Append
    Write-Host "Exported to CSV successfully!"  -ForegroundColor Green	

}

Function StartProcessing {
    Login($creds);
    GetOnedriveSitesDeails
}

StartProcessing
```
[!INCLUDE [More about SPO Management Shell](../../docfx/includes/MORE-SPOMS.md)]
***


## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/export-onedrive-sites-details-to-csv" aria-hidden="true" />

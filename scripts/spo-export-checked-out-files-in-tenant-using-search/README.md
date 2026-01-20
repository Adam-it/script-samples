

# Getting checked-out files in the tenant using Search

## Summary

It requires a lot of work to iterate all Site Collections looking for checked out files. If you can accept that the quality is slightly lower ( some sites or libraries might be excluded from Search) this script can provide the list of checkout file in minutes, not hours

![Example Screenshot](assets/example.png)


# [PnP PowerShell](#tab/pnpps)

```powershell

#Config variables
$tenantAdminURL = "https://contoso-admin.sharepoint.com/"
$reportOutput = "C:\Temp\CheckedOutFiles.csv"


$tenantconn = Connect-PnPOnline -Url $tenantAdminURL -ClientId "YOURCLIENTID" -ClientSecret "YOURCLIENTSECRET"  -ReturnConnection


function Get-CheckedOutItems ($emaildomain)
{
    $query = "CheckoutUserOWSUSER:"+$emaildomain
    $searchres= Invoke-PnPSearchQuery -Query $query -All -Connection $tenantconn -SelectProperties Path,CheckoutUserOWSUSER,LastModifiedTime
    $searchres.ResultRows.Count
    $checkedoutitems= @()
    $index = 0
    foreach($row in $searchres.ResultRows)
    {
        $CheckedOutToName = $row["CheckoutUserOWSUSER"].substring($row["CheckoutUserOWSUSER"].IndexOf("|")+1)
        $CheckedOutToName = $CheckedOutToName.Substring(0,$CheckedOutToName.IndexOf("|"))
        $CheckedOutToName= $CheckedOutToName.Trim()

        $CheckedOutToEmail = $row["CheckoutUserOWSUSER"].substring(0,$row["CheckoutUserOWSUSER"].IndexOf("|")-1)

        Write-Host "$index of $($searchres.ResultRows.Count)" -ForegroundColor Green
        $Data = New-Object PSObject
        $Data | Add-Member NoteProperty Title($row.Title) 
        
        $Data | Add-Member NoteProperty CheckedOutToName($CheckedOutToName)
        $Data | Add-Member NoteProperty CheckedOutToEmail($CheckedOutToEmail)
        $Data | Add-Member NoteProperty URL($row["Path"]) 
        $Data | Add-Member NoteProperty LastModified($row["LastModifiedTime"]) 
        
        $checkedoutitems += $Data
        $index++
    }
    
    $checkedoutitems | Export-Csv -Path $reportOutput -Encoding utf8BOM -Force -Delimiter "¤"
    
        
}

#Can either be generic like "contoso.com" or specific like "john.doe@contoso.com"
Get-CheckedOutItems -emaildomain "contoso.com"


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $false, HelpMessage = "Email domain or specific user email to search (e.g., 'contoso.com' or 'john.doe@contoso.com'). Use '*' for all checked-out files.")]
    [string]$EmailDomain = "*",

    [Parameter(Mandatory = $false, HelpMessage = "Output path for CSV report")]
    [string]$OutputPath = (Get-Location).Path,

    [Parameter(Mandatory = $false, HelpMessage = "Batch size for search results (default: 500)")]
    [ValidateRange(1, 500)]
    [int]$BatchSize = 500,

    [Parameter(Mandatory = $false, HelpMessage = "Tenant admin URL for enhanced search context")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$TenantAdminUrl
)

begin {
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to Microsoft 365. Please run 'm365 login' first."
    }

    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath -PathType Container)) {
            throw "Output path '$OutputPath' does not exist or is not a directory."
        }
    }

    $script:ReportCollection = @()
    $script:Summary = @{FilesFound = 0; ParseErrors = 0}

    $transcriptPath = "$OutputPath\CheckedOutFiles_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    Start-Transcript -Path $transcriptPath
}

process {
    try {
        Write-Host "Searching for checked-out files..." -ForegroundColor Cyan
        if ($EmailDomain -ne "*") {
            Write-Host "  Filtering by email domain: $EmailDomain" -ForegroundColor White
        } else {
            Write-Host "  Searching across entire tenant" -ForegroundColor White
        }

        if ($EmailDomain -eq "*") {
            $query = "CheckoutUserOWSUSER:*"
        } else {
            $query = "CheckoutUserOWSUSER:$EmailDomain"
        }

        $searchArgs = @(
            'spo', 'search',
            '--queryText', $query,
            '--selectProperties', 'Path,CheckoutUserOWSUSER,LastModifiedTime,Title',
            '--allResults',
            '--rowLimit', $BatchSize,
            '--output', 'json'
        )

        if ($TenantAdminUrl) {
            $searchArgs += '--webUrl'
            $searchArgs += $TenantAdminUrl
        }

        $resultsJson = m365 @searchArgs 2>&1

        if ($LASTEXITCODE -ne 0) {
            throw "Search query failed: $resultsJson"
        }

        $results = @($resultsJson | ConvertFrom-Json)
        $script:Summary.FilesFound = $results.Count

        Write-Host "  Found $($results.Count) checked-out files" -ForegroundColor Green

        foreach ($item in $results) {
            try {
                $checkoutUser = $item.CheckoutUserOWSUSER

                if ($checkoutUser -match "^([^|]+)\\|([^|]+)\\|") {
                    $email = $matches[1]
                    $name = $matches[2].Trim()
                } else {
                    Write-Warning "Failed to parse user from: $checkoutUser"
                    $email = "Unknown"
                    $name = $checkoutUser
                    $script:Summary.ParseErrors++
                }

                $script:ReportCollection += [PSCustomObject]@{
                    Title = $item.Title ?? "Unknown"
                    CheckedOutToName = $name
                    CheckedOutToEmail = $email
                    URL = $item.Path
                    LastModified = $item.LastModifiedTime
                }
            }
            catch {
                Write-Warning "Failed to process item: $($_.Exception.Message)"
                $script:Summary.ParseErrors++
            }
        }
    }
    catch {
        Write-Error "Search failed: $($_.Exception.Message)"
        throw
    }
}

end {
    Stop-Transcript

    if ($script:ReportCollection.Count -gt 0) {
        $csvPath = "$OutputPath\CheckedOutFiles_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "\nCSV report saved: $csvPath" -ForegroundColor White
    } else {
        Write-Host "\nNo checked-out files found." -ForegroundColor Yellow
    }

    Write-Host "\n===== Summary =====" -ForegroundColor Cyan
    Write-Host "Files Found: $($script:Summary.FilesFound)" -ForegroundColor Green
    $parseColor = if ($script:Summary.ParseErrors -gt 0) { "Yellow" } else { "Green" }
    Write-Host "Parse Errors: $($script:Summary.ParseErrors)" -ForegroundColor $parseColor
}

# Example: Search all checked-out files in tenant
# .\Export-CheckedOutFiles.ps1

# Example: Search for specific user
# .\Export-CheckedOutFiles.ps1 -EmailDomain "john.doe@contoso.com"

# Example: Search by domain with custom output path
# .\Export-CheckedOutFiles.ps1 -EmailDomain "contoso.com" -OutputPath "C:\Reports"

# Example: Search with tenant admin URL for broader scope and custom batch size
# .\Export-CheckedOutFiles.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -BatchSize 100 -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***


## Contributors

| Author(s) |
|-----------|
| Adam Wójcik |
| Kasper Larsen |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-checked-out-files-in-tenant-using-search" aria-hidden="true" />

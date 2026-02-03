

# Get Site Ids to URL

## Summary

Converts unique site IDs from a txt file to URLs using Microsoft Search for M365 Tenancy and exports to CSV. Available using PnP PowerShell or CLI for Microsoft 365.

![Example Screenshot](assets/example.png)

This PowerShell script takes an input file containing one or more SharePoint online (Office 365) Site Collection Object IDs and converts them into the full URLs. It requires PnP Online module for connection to Office 365, performs a search query using these GUIDs as parameters, retrieves site details including their respective URL addresses from each result row.

Note: Above description uses AI to describe the script.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Path to input text file containing Site IDs (one per line)")]
    [ValidateScript({
        if (Test-Path -Path $_ -PathType Leaf) {
            $true
        } else {
            throw "Input file '$_' does not exist."
        }
    })]
    [string]$InputFile,

    [Parameter(Mandatory = $false, HelpMessage = "Output directory for CSV export")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Host "Authenticating to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate to Microsoft 365. Please check your credentials."
    }

    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $csvPath = Join-Path -Path $OutputPath -ChildPath "SiteIdToURL_$timestamp.csv"

    $script:Results = [System.Collections.Generic.List[PSCustomObject]]::new()
    $script:Summary = @{
        Total     = 0
        Found     = 0
        NotFound  = 0
        Failures  = 0
    }

    $siteIds = Get-Content -Path $InputFile | Where-Object { $_.Trim() -ne "" }
    $script:Summary.Total = $siteIds.Count

    Write-Host "Found $($script:Summary.Total) site ID(s) in input file" -ForegroundColor White
    Write-Host "Starting site ID lookup...`n" -ForegroundColor Cyan
}

process {
    $count = 0

    foreach ($siteId in $siteIds) {
        $count++
        Write-Progress -Activity "Processing site IDs" -Status "$count of $($script:Summary.Total): $siteId" -PercentComplete (($count / $script:Summary.Total) * 100)

        try {
            $query = "SiteId:$siteId contentClass:STS_Site"
            $result = m365 spo search --queryText $query --selectProperties "Title,Path,SiteId,WebTemplate" --output json 2>&1 | Out-String

            if ($LASTEXITCODE -ne 0) {
                throw "Search command failed with exit code $LASTEXITCODE: $result"
            }

            $searchResults = $result | ConvertFrom-Json

            if ($searchResults -and $searchResults.Count -gt 0) {
                foreach ($site in $searchResults) {
                    $script:Results.Add([PSCustomObject]@{
                        SiteId      = $siteId
                        Title       = $site.Title
                        Path        = $site.Path
                        WebTemplate = $site.WebTemplate
                    })
                }
                Write-Verbose "Found site: $($searchResults[0].Path)"
                $script:Summary.Found++
            } else {
                Write-Warning "Site ID not found: $siteId"
                $script:Results.Add([PSCustomObject]@{
                    SiteId      = $siteId
                    Title       = "NOT FOUND"
                    Path        = "NOT FOUND"
                    WebTemplate = "NOT FOUND"
                })
                $script:Summary.NotFound++
            }
        }
        catch {
            Write-Warning "Failed to process site ID '$siteId': $($_.Exception.Message)"
            $script:Results.Add([PSCustomObject]@{
                SiteId      = $siteId
                Title       = "ERROR"
                Path        = "ERROR: $($_.Exception.Message)"
                WebTemplate = "ERROR"
            })
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Progress -Activity "Processing site IDs" -Completed

    if ($script:Results.Count -gt 0) {
        $script:Results | Export-Csv -Path $csvPath -NoTypeInformation -Force
        Write-Host "`nCSV export completed: $csvPath" -ForegroundColor Green
    } else {
        Write-Warning "No results to export."
    }

    Write-Host "`n===== Summary =====" -ForegroundColor Cyan
    Write-Host "Input file: $InputFile" -ForegroundColor White
    Write-Host "Output CSV: $csvPath" -ForegroundColor White
    Write-Host "Total site IDs processed: $($script:Summary.Total)" -ForegroundColor White
    Write-Host "Found: $($script:Summary.Found)" -ForegroundColor Green

    if ($script:Summary.NotFound -gt 0) {
        Write-Host "Not found: $($script:Summary.NotFound)" -ForegroundColor Yellow
    } else {
        Write-Host "Not found: $($script:Summary.NotFound)" -ForegroundColor White
    }

    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor White
    }
}

# Example 1: Convert site IDs from input file
# .\Get-SiteIdToURL.ps1 -InputFile "C:\Temp\SiteIDs.txt"

# Example 2: Specify custom output directory
# .\Get-SiteIdToURL.ps1 -InputFile "C:\Temp\SiteIDs.txt" -OutputPath "C:\Reports"

# Example 3: Run with verbose output to see each site as it's found
# .\Get-SiteIdToURL.ps1 -InputFile "C:\Temp\SiteIDs.txt" -Verbose

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

# [PnP PowerShell](#tab/pnpps)

```powershell

param(
    [Parameter(Mandatory=$true)]
    [ValidatePattern("^https://.*\.sharepoint\.com$", ErrorMessage="Please enter a valid SharePoint Online Admin URL")]
    [string]$SPOAdminURL = $(Read-Host -Prompt "Please enter the SharePoint Online Admin URL"),
   
    [Parameter(Mandatory=$false)]
    [string]$log = ".\SPOSiteURLs.csv"
)

## Load Form Selector
Add-Type -AssemblyName System.Windows.Forms

function Select-FileDialog {
    param([string]$Title, [string]$Directory, [string]$Filter="All Files (*.*)|*.*")
    $objForm = New-Object System.Windows.Forms.OpenFileDialog
    $objForm.InitialDirectory = $Directory
    $objForm.Filter = $Filter
    $objForm.Title = $Title
    $Show = $objForm.ShowDialog()
    if ($Show -eq "OK") {
        return $objForm.FileName
    } else {
        Write-Error "Operation cancelled by user."
        exit
    }
}

## Check Execution Policy
$currentPolicy = Get-ExecutionPolicy
if ($currentPolicy -ne "Unrestricted") {
    Write-Host "Current execution policy is $currentPolicy. This script requires it to be Unrestricted." -ForegroundColor Yellow
    try {
        Set-ExecutionPolicy Unrestricted -Scope Process -Force
        Write-Host "Execution policy set to Unrestricted for this session." -ForegroundColor Green
    } catch {
        Write-Error "Failed to set execution policy. Please run PowerShell as an administrator and try again."
        exit
    }
}

## Check to see if PNP is installed
Write-Host "Please ensure that you are running PowerShell in Admin Mode" -ForegroundColor Yellow
$CheckPNP = Get-Module -Name PnP.PowerShell -ListAvailable
if ($CheckPNP -eq $null) {
    Write-Host "It appears you do not have SharePoint Online PNP installed!" -ForegroundColor Red
    $Force = Read-Host "Would you like to install SharePoint Online PNP Module? Type 'Y' to force or type 'N' to continue"
    if ($Force -like "y") {
        try {
            Install-Module -Name PnP.PowerShell -Force -ErrorAction Stop
            Import-Module PnP.PowerShell -ErrorAction Stop
            Write-Host "PnP PowerShell module installed successfully." -ForegroundColor Green
        } catch {
            Write-Error "Failed to install PnP PowerShell module. Exiting script."
            exit
        }
    } elseif ($Force -like "n") {
        Write-Host "Continuing without install of PNP and assuming module was not detected properly" -ForegroundColor Yellow
    }
}

# Select Input File
Write-Host "Please select the input text file which has the site collection GUID's......." -ForegroundColor DarkGreen
$InputFile = Select-FileDialog -Title "Select the input file of site ID's to convert to URL's"

if (-not (Test-Path -Path $InputFile -PathType Leaf)) {
    Write-Error "The selected file does not exist. Exiting script."
    exit
}

$cnt = Get-Content $InputFile

$starttime = Get-Date

## Connect to PNP PowerShell
try {
    Connect-PnPOnline -Url $SPOAdminURL -Interactive -ErrorAction Stop
} catch {
    Write-Error "Failed to connect to SharePoint Online. Please check your admin URL and credentials."
    exit
}

## Create Result Set
[System.Collections.Generic.List[PSCustomObject]] $results = New-Object System.Collections.Generic.List[PSCustomObject]

$count = 0

foreach ($siteid in $cnt) {
    try {
        Write-Progress -Activity 'Processing sites..' -Status $siteid -PercentComplete ($count / $cnt.count * 100)
        $query = "SiteId:$siteid contentClass:STS_Site"
        $result = Submit-PnPSearchQuery -Query $query -ErrorAction Stop
        foreach ($row in $result.ResultRows) {
            $res = New-Object psobject
            foreach ($key in $row.Keys) {
                $res | Add-Member -MemberType NoteProperty -Name $key -Value $row[$key]
            }
            $results.Add($res)
        }
        $count++
    } catch {
        Write-Host "Failed to process $siteid. Error: $_" -ForegroundColor Red
        continue
    }
}

try {
    $results | Export-Csv -Path $log -NoTypeInformation -Force -Append
} catch {
    Write-Error "Failed to export results to CSV. Error: $_"
    exit
}

$duration = (Get-Date) - $starttime
Write-Host "`nComplete in $($duration)!" -ForegroundColor Green
Write-Host "Total sites processed: $count" -ForegroundColor Cyan
Write-Host "Please review the log at $($log)" -ForegroundColor Cyan 

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Source

This script was first created on PnP PowerShell and transferred over in Dec 2024.
https://github.com/pnp/powershell


## Contributors

| Author(s) |
|-----------|
| Sam Larson |
| Paul Bullock |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-site-list-ids" aria-hidden="true" />

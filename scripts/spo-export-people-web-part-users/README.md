

# Sample showing how to extract the employees shown in the People Web part on pages in a selection of Site Collections to CSV

## Summary

One of my customs requested that we removed a specific employee from the People Web part in their Intranet ASAP as that employee had left rather abruptly. They had no idea where that employee was displayed hence this script. This script scans Communication sites for pages containing People web parts and exports employee information to CSV.

## Implementation

- Open VS Code
- Create a new file
- Copy the code below,
- Change the variables to target to your environment
- Run the script.
 
## Screenshot of Output 

![Example Screenshot](assets/preview.png)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory = $false, HelpMessage = "Filter sites by template (e.g., 'Template eq ''STS#3''' for Communication sites)")]
    [string]$SiteFilter = "Template eq 'STS#3'",
    
    [Parameter(Mandatory = $false, HelpMessage = "Path where the CSV report will be saved")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = Join-Path $OutputPath "PeopleWebPartUsers_$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Starting People Web Part User Extraction..." -ForegroundColor Cyan
    
    # Initialize report collection
    $script:ReportCollection = @()
    
    # Initialize counters
    $script:SitesProcessed = 0
    $script:PagesProcessed = 0
    $script:PeopleFound = 0
    $script:Failures = 0
    
    # Ensure user is logged in
    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to Microsoft 365. Please run 'm365 login' first."
    }
    Write-Host "Successfully authenticated" -ForegroundColor Green
    
    # Validate output path
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath -PathType Container)) {
            throw "Output path '$OutputPath' does not exist or is not a directory."
        }
    }
    
    # Get filtered site collections
    Write-Host "Retrieving site collections (Filter: $SiteFilter)..." -ForegroundColor Cyan
    $sitesJson = m365 spo site list --filter "$SiteFilter" --output json
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve site collections. Error: $sitesJson"
    }
    
    $script:Sites = @($sitesJson | ConvertFrom-Json)
    Write-Host "Found $($script:Sites.Count) site(s) to process" -ForegroundColor Yellow
}

process {
    foreach ($site in $script:Sites) {
        $script:SitesProcessed++
        Write-Host "`nProcessing site: $($site.Title) ($($site.Url))" -ForegroundColor Cyan
        
        try {
            # Get all pages in the site
            Write-Verbose "Retrieving pages from site: $($site.Url)"
            $pagesJson = m365 spo page list --webUrl $site.Url --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve pages from site: $($site.Url). Skipping..."
                $script:Failures++
                continue
            }
            
            $pages = @($pagesJson | ConvertFrom-Json)
            Write-Verbose "Found $($pages.Count) page(s) in site"
            
            # Process each page
            foreach ($page in $pages) {
                $script:PagesProcessed++
                Write-Verbose "Scanning page: $($page.Name)"
                
                try {
                    # Get page controls
                    $controlsJson = m365 spo page control list --webUrl $site.Url --pageName $page.Name --output json
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to retrieve controls from page: $($page.Name). Skipping..."
                        $script:Failures++
                        continue
                    }
                    
                    $controls = @($controlsJson | ConvertFrom-Json)
                    
                    # Filter People web parts (check if controlData.webPartData.properties.persons exists)
                    foreach ($control in $controls) {
                        if ($control.controlData.webPartData.properties.PSObject.Properties['persons']) {
                            $persons = $control.controlData.webPartData.properties.persons
                            Write-Host "  Found People web part with $($persons.Count) person(s) on page: $($page.Title)" -ForegroundColor Green
                            
                            foreach ($person in $persons) {
                                $script:PeopleFound++
                                
                                # Extract person ID (strip i:0#.f|membership| prefix if present)
                                $personId = $person.Id
                                if ($personId -and $personId.IndexOf("i:0#.f|membership|") -gt -1) {
                                    $personId = $personId.Substring(18)
                                }
                                
                                # Add to report
                                $script:ReportCollection += [PSCustomObject]@{
                                    SiteUrl = $site.Url
                                    SiteTitle = $site.Title
                                    PageUrl = $page.AbsoluteUrl
                                    PageTitle = $page.Title
                                    PersonId = $personId
                                    PersonUpn = $person.upn
                                    ErrorCode = ""
                                }
                            }
                        }
                    }
                }
                catch {
                    Write-Warning "Error processing page '$($page.Name)': $($_.Exception.Message)"
                    $script:Failures++
                    
                    # Add error entry to report
                    $script:ReportCollection += [PSCustomObject]@{
                        SiteUrl = $site.Url
                        SiteTitle = $site.Title
                        PageUrl = if ($page.AbsoluteUrl) { $page.AbsoluteUrl } else { "$($site.Url)/SitePages/$($page.Name)" }
                        PageTitle = $page.Title
                        PersonId = ""
                        PersonUpn = ""
                        ErrorCode = $_.Exception.Message
                    }
                }
            }
        }
        catch {
            Write-Warning "Error processing site '$($site.Url)': $($_.Exception.Message)"
            $script:Failures++
        }
    }
}

end {
    # Export report to CSV
    $csvPath = Join-Path $OutputPath "PeopleWebPartUsers_$timestamp.csv"
    
    if ($script:ReportCollection.Count -gt 0) {
        $script:ReportCollection | Export-Csv -Path $csvPath -Encoding UTF8 -NoTypeInformation -Force
        Write-Host "`nReport exported to: $csvPath" -ForegroundColor Green
    }
    else {
        Write-Host "`nNo People web parts found. No report generated." -ForegroundColor Yellow
    }
    
    # Display summary
    Write-Host "`n===== Execution Summary =====" -ForegroundColor Cyan
    Write-Host "Sites Processed   : $script:SitesProcessed" -ForegroundColor White
    Write-Host "Pages Scanned     : $script:PagesProcessed" -ForegroundColor White
    Write-Host "People Found      : $script:PeopleFound" -ForegroundColor White
    
    if ($script:Failures -gt 0) {
        Write-Host "Failures          : $script:Failures" -ForegroundColor Red
    }
    else {
        Write-Host "Failures          : $script:Failures" -ForegroundColor Green
    }
    
    Write-Host "Transcript        : $transcriptPath" -ForegroundColor White
    Write-Host "============================" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Example 1: Export People web part users from all Communication sites
# .\Export-PeopleWebPartUsers.ps1

# Example 2: Export from custom site template with verbose output
# .\Export-PeopleWebPartUsers.ps1 -SiteFilter "Template eq 'SITEPAGEPUBLISHING#0'" -Verbose

# Example 3: Save report to custom location
# .\Export-PeopleWebPartUsers.ps1 -OutputPath "C:\Reports" -Verbose

# Example 4: Export from all modern sites (Team + Communication)
# .\Export-PeopleWebPartUsers.ps1 -SiteFilter "Template eq 'GROUP#0' or Template eq 'STS#3'" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell

# Author Kasper Larsen Fellowmind.dk
# Purpose : locate any People Web part and report the people displayed


#define which site collections you wish to iterate
$tenentUrl = "https://[tenant].sharepoint.com"
$relevantsitecollections = Get-PnPTenantSite | Where-Object {$_.template -eq "STS#3"}


$Output = @()


foreach($site in  $relevantsitecollections)
{
    $sitecollectionUrl = $site.Url
    
    Write-Host "Url =  $sitecollectionUrl" -ForegroundColor Yellow
    
    Connect-PnPOnline -Url $sitecollectionUrl -Interactive
    $pages = Get-PnPListItem -List "sitePages" 
    
    foreach($page in $pages)
    {
        try 
        {
            $fullUrl = $tenentUrl+$page["FileRef"]
            Write-Host " Page = $fullUrl" -ForegroundColor Green
            $webpartpage = Get-PnPClientSidePage -Identity $page["FileLeafRef"] -ErrorAction Stop
            
            $webparts = $webpartpage.controls | Where-Object {$_.PropertiesJson -like "*persons*"}

            foreach($webpart in $webparts)
            {
                $props =  $webpart.PropertiesJson | ConvertFrom-Json
                write-host "Found $props.persons.count people in the web part" -ForegroundColor Blue
                foreach($person in $props.persons)
                {
                    $personId = $person.Id
                    if($personId.IndexOf("i:0#.f|membership|") -gt -1)
                    {
                        $personId = $personId.substring(18)
                    }

                    $myObject = [PSCustomObject]@{
                        URL     = $tenentUrl+$page["FileRef"]
                        personid = $personId
                        personupn = $person.upn
                        errorcode = ""

                    }        
                    $Output+=($myObject)
                }
            }
        }
        catch 
        {
            $myObject = [PSCustomObject]@{
                URL     = $tenentUrl+$page["FileRef"]
                personid = ""
                personupn = ""
                errorcode = $_.Exception.Message

            }        
            $Output+=($myObject)
        }
        
        
        
        
    }
}
$Output | Export-Csv  -Path c:\temp\PeopleWebPartUsers.csv -Encoding utf8NoBOM -Force  -Delimiter "|"

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| Kasper Larsen, Fellowmind|
| Adam Wójcik, Adam-it |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-basic-sitecollection-info" aria-hidden="true" />

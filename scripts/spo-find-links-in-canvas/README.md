

# Find Links in Modern Page

## Summary

Script will take a CSV file that contains URLs to SharePoint sites and analyze the site pages to see if any of the pages have hyperlinks.
For every hyperlink in a page this gets output to a row in a csv that is delimited by a pipe

The script reads a list of SharePoint sites from a CSV file, connects to each site and extracts all pages within the specified lists. For every page content that contains an anchor tag, it captures both 'Title' field value (from Page metadata) along with any href tags present in its body text using regex matching, then writes these details into a new CSV file named after today’s date.

Note: Above last paragraph of description uses AI to describe the script.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage="Path to CSV file containing site URLs with 'Url' column")]
    [string]$SitesCSV,
    
    [Parameter(HelpMessage="Output directory path for the CSV report. Defaults to current location")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    # Start transcript for logging
    $timestamp = Get-Date -Format "yyyyMMddHHmmss"
    $transcriptPath = Join-Path $OutputPath "FindLinksInCanvas_$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Starting link extraction from modern pages..." -ForegroundColor Cyan
    
    # Ensure user is logged in to Microsoft 365
    Write-Verbose "Ensuring Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365. Please run 'm365 login' first."
    }
    Write-Verbose "Successfully authenticated to Microsoft 365"
    
    # Validate CSV file exists
    if (-not (Test-Path $SitesCSV)) {
        throw "CSV file not found at path: $SitesCSV"
    }
    
    # Validate OutputPath exists
    if ($PSBoundParameters.ContainsKey('OutputPath') -and -not (Test-Path $OutputPath)) {
        throw "Output path does not exist: $OutputPath"
    }
    
    # Initialize collections and counters
    $script:Results = @()
    $script:TotalSites = 0
    $script:TotalPages = 0
    $script:TotalLinks = 0
    $script:FailedPages = 0
    
    # Regex pattern to extract href tags
    $script:HrefRegex = '<a\s+(?:[^>]*?\s+)?href=(["''])(.*?)\1>'
    
    Write-Verbose "Initialization complete. Reading sites from CSV..."
}

process {
    # Read CSV file
    $sites = Import-Csv -Path $SitesCSV
    
    $siteIndex = 0
    $totalSites = $sites.Count
    
    foreach ($site in $sites) {
        $siteUrl = $site.Url
        $script:TotalSites++
        $siteIndex++
        
        Write-Progress -Activity "Scanning sites for links" `
            -Status "Processing site $siteIndex of $totalSites: $siteUrl" `
            -PercentComplete (($siteIndex / $totalSites) * 100) `
            -Id 1
        
        Write-Host "Processing site: $siteUrl" -ForegroundColor Yellow
        
        try {
            # Get site information for title
            Write-Verbose "Retrieving site details for: $siteUrl"
            $webJson = m365 spo web get --url $siteUrl --output json
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve site details"
            }
            $web = $webJson | ConvertFrom-Json
            $siteTitle = $web.Title
            Write-Verbose "Site title: $siteTitle"
            
            # Get all pages from the site
            Write-Verbose "Retrieving pages from: $siteUrl"
            $pagesJson = m365 spo page list --webUrl $siteUrl --output json
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve pages"
            }
            $pages = @($pagesJson | ConvertFrom-Json)
            Write-Verbose "Found $($pages.Count) pages"
            
            $pageIndex = 0
            $totalPages = $pages.Count
            
            foreach ($page in $pages) {
                $script:TotalPages++
                $pageIndex++
                
                Write-Progress -Activity "Scanning pages in site" `
                    -Status "Processing page $pageIndex of $totalPages: $($page.Title)" `
                    -PercentComplete (($pageIndex / $totalPages) * 100) `
                    -Id 2 -ParentId 1
                
                try {
                    # Get canvas content from page
                    $canvasContent = $page.CanvasContent1
                    
                    # Skip pages with no canvas content
                    if ([string]::IsNullOrEmpty($canvasContent)) {
                        Write-Verbose "Skipping page '$($page.Title)' - no canvas content"
                        continue
                    }
                    
                    # Extract href tags using regex
                    $hrefMatches = [regex]::Matches($canvasContent, $script:HrefRegex)
                    
                    if ($hrefMatches.Count -gt 0) {
                        Write-Verbose "Found $($hrefMatches.Count) links in page '$($page.Title)'"
                        
                        foreach ($match in $hrefMatches) {
                            $hrefTag = $match.Value
                            $script:TotalLinks++
                            
                            # Add to results collection
                            $script:Results += [PSCustomObject]@{
                                SiteTitle = $siteTitle
                                PageTitle = $page.Title
                                PageUrl   = $page.AbsoluteUrl
                                HrefTag   = $hrefTag
                            }
                        }
                    } else {
                        Write-Verbose "No links found in page '$($page.Title)'"
                    }
                }
                catch {
                    $script:FailedPages++
                    Write-Warning "Failed to process page '$($page.Title)': $($_.Exception.Message)"
                    continue
                }
            }
        }
        catch {
            Write-Warning "Failed to process site '$siteUrl': $($_.Exception.Message)"
            continue
        }
    }
    
    # Clear progress bars
    Write-Progress -Activity "Scanning sites for links" -Id 1 -Completed
    Write-Progress -Activity "Scanning pages in site" -Id 2 -Completed
}

end {
    # Export results to CSV with pipe delimiter
    if ($script:Results.Count -gt 0) {
        $outputFile = Join-Path $OutputPath "LinkMatches_$timestamp.csv"
        
        # Export with pipe delimiter (matching PnP format)
        $script:Results | Export-Csv -Path $outputFile -Delimiter '|' -NoTypeInformation
        
        Write-Host "`nResults exported to: $outputFile" -ForegroundColor Green
    } else {
        Write-Host "`nNo links found in any pages" -ForegroundColor Yellow
    }
    
    # Display summary
    Write-Host "`n==== Summary ====" -ForegroundColor Cyan
    Write-Host "Sites scanned: $($script:TotalSites)" -ForegroundColor White
    Write-Host "Pages analyzed: $($script:TotalPages)" -ForegroundColor White
    Write-Host "Links found: $($script:TotalLinks)" -ForegroundColor White
    
    if ($script:FailedPages -gt 0) {
        Write-Host "Failed pages: $($script:FailedPages)" -ForegroundColor Red
    } else {
        Write-Host "Failed pages: $($script:FailedPages)" -ForegroundColor Green
    }
    
    # Stop transcript
    Write-Host "`nTranscript saved to: $transcriptPath" -ForegroundColor Cyan
    Stop-Transcript
}

# Usage examples:

# Example 1: Basic usage with CSV file
# .\FindLinksInCanvas.ps1 -SitesCSV "C:\Temp\sites.csv"

# Example 2: Specify custom output path
# .\FindLinksInCanvas.ps1 -SitesCSV "C:\Temp\sites.csv" -OutputPath "C:\Reports"

# Example 3: Run with verbose output
# .\FindLinksInCanvas.ps1 -SitesCSV "C:\Temp\sites.csv" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell

 # Each site in this list will have the script run against
$csv_SiteList = "sites-test.csv"
$csv_siteheaders = 'Url'

# Date used in the file creation
$date = Get-Date
$date = $date.ToString("yyyymmddhhss")

# filename by using the date
$file_name = $date + 'LinkMatches.csv'

# Path to create the output fil
$creation_path = Get-Location

# The site pages list that this script will run against
$List = "SitePages"

# Headers for the output csv
$headers = "Site Title|Page Title|Page Url|Href Tag"

# new line character
$ofs = "`n"

# delimiter to use
$delim = '|'

# regex used to match the href tags that are embeded in the canvas page content
$regex ='<a\s+(?:[^>]*?\s+)?href=(["])(.*?)\1>'

# create object of all the sites
$sites = Import-Csv -Path $csv_SiteList -Header $csv_siteheaders

#variable for the header
$csv_outputheader = $headers + $ofs

#complete file path
$csv_path = Join-Path $creation_path $file_name

# create output csv
New-Item -Path $creation_path -Name $file_name -ItemType File -Value $csv_outputheader

# itterate around each site from the csv
foreach($site in $sites)
{
    # make the connection, get ome site information and create object that contains all the site pages
    $connection = Connect-PnPOnline -Url $site.Url -Interactive
    $pnpsite = Get-PnPWeb -Connection $connection
    $site_title = $pnpsite.Title
    $pages = (Get-PnPListItem -List $List -Fields "CanvasContent1", "Title" -Connection $connection).FieldValues

    # itterate around each page in the stie to get the information from each page that will be used to build up the row and also conduct
    # the check to see if the canvas content has any href tags embeded
    foreach($page in $pages)
    {
        $page_title = $page.Get_Item("Title")
        $fileref = $page.Get_Item("FileRef")
        $canvascontent = $page.Get_Item("CanvasContent1")
        # check if the canvas has content 
        if ($canvascontent.Length -gt 0) 
        {
            # hash table of the results that match the href regular expression
            $hrefmatches = ($canvascontent | select-string -pattern $regex -AllMatches).Matches.Value

            # itterate around each regular expression match and write it out into the output csv that is pipe delimited 
            foreach($hrefmatch in $hrefmatches)
            {
                $row = $site_title + $delim + $page_title + $delim + $fileref + $delim + $hrefmatch
                Add-Content -Path $csv_path -Value $row
            }
        }
    }
    Disconnect-PnPOnline
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Source

This script was first created on PnP PowerShell and transferred over in Dec 2024. Details of the orignal author missing. Report if inaccurate.
https://github.com/pnp/powershell

## Contributors

| Author(s) |
|-----------|
| Paul Bullock |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-find-links-in-canvas" aria-hidden="true" />

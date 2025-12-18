

# Sample showing how to Extract the employees shown on modern pages (Author byline) in a selection of Site Collections to CSV

## Summary

One of my customs requested that we removed a specific employee from the modern pages in their Intranet ASAP as that employee had left rather abruptly. They had no idea where that employee was displayed hence this script.

This sample is available in both PnP PowerShell and CLI for Microsoft 365 versions.

## Implementation

- Open VS Code
- Create a new file
- Copy the code below,
- Change the variables to target to your environment
- Run the script.
 
## Screenshot of Output

![Example Screenshot](assets/preview.png)

# [PnP PowerShell](#tab/pnpps)
```powershell

# Author Kasper Larsen Fellowmind.dk
# Purpose : locate any Author byline and report the people displayed


$tenentUrl = "https://[Tenant].sharepoint.com"

Connect-PnPOnline -Url $tenentUrl -Interactive

#define which site collections you wish to iterate
#$relevantsitecollections = Get-PnPTenantSite | Where-Object {$_.template -eq "STS#3"}
$relevantsitecollections = Get-PnPTenantSite 

$Output = @()

foreach($site in  $relevantsitecollections)
{
    $sitecollectionUrl = $site.Url
    Write-Host "Url =  $sitecollectionUrl" -ForegroundColor Yellow
    Connect-PnPOnline -Url $sitecollectionUrl -Interactive
    $pages = Get-PnPListItem -List "sitePages" -ErrorAction SilentlyContinue
    
    foreach($page in $pages)
    {
        try 
        {
            $authorbyline = $page["_AuthorByline"]
            if($authorbyline)
            {
                $myObject = [PSCustomObject]@{
                    URL     = $tenentUrl+$page["FileRef"]
                    Email = $authorbyline.Email
                    Name = $authorbyline.LookupValue
                    errorcode = ""

                }        
                $Output+=($myObject)
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
$Output | Export-Csv  -Path c:\temp\AuthorbylineUsers.csv -Encoding utf8BOM -Force  -Delimiter "|"

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Export-AuthorBylineUsers {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory = $true, HelpMessage = "The URL of the SharePoint tenant")]
        [string]$TenantUrl,

        [Parameter(HelpMessage = "OData filter to limit which sites to process")]
        [string]$SiteFilter,

        [Parameter(HelpMessage = "The directory path where the CSV file will be saved")]
        [string]$OutputPath = (Get-Location).Path,

        [Parameter(HelpMessage = "Export results to a CSV file")]
        [switch]$ExportToCsv
    )

    begin {
        # Login to Microsoft 365
        Write-Verbose "Ensuring Microsoft 365 login..."
        m365 login --ensure 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to authenticate to Microsoft 365. Please run 'm365 login' first."
        }
        Write-Verbose "Successfully authenticated to Microsoft 365"

        # Initialize summary and collection
        $script:Summary = @{
            SitesProcessed     = 0
            PagesFound         = 0
            AuthorBylinesFound = 0
            Failures           = 0
        }
        $script:AuthorBylineCollection = @()
    }

    process {
        try {
            # Get all sites
            Write-Host "Retrieving SharePoint sites..." -ForegroundColor Cyan
            
            if ($SiteFilter) {
                Write-Verbose "Using site filter: $SiteFilter"
                $sitesJson = m365 spo site list --filter "$SiteFilter" --output json 2>&1
            }
            else {
                $sitesJson = m365 spo site list --output json 2>&1
            }
            
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve sites. CLI output: $sitesJson"
            }

            $sites = @($sitesJson | ConvertFrom-Json)
            Write-Host "Found $($sites.Count) sites to process" -ForegroundColor Cyan

            # Process each site
            foreach ($site in $sites) {
                $siteUrl = $site.Url
                Write-Host "Processing site: $siteUrl" -ForegroundColor Yellow
                $script:Summary.SitesProcessed++

                try {
                    # Get pages from Site Pages library
                    # Using --listTitle with error handling as some sites may not have Site Pages library
                    Write-Verbose "Retrieving pages from Site Pages library..."
                    $pagesJson = m365 spo listitem list --webUrl $siteUrl --listTitle "Site Pages" --fields "ID,FileRef,_AuthorByline/Email,_AuthorByline/LookupValue" --output json 2>&1
                    
                    if ($LASTEXITCODE -ne 0) {
                        Write-Verbose "Site Pages library not found or inaccessible on $siteUrl (this is normal for some sites)"
                        continue
                    }

                    $pages = @($pagesJson | ConvertFrom-Json)
                    $script:Summary.PagesFound += $pages.Count
                    Write-Verbose "Found $($pages.Count) pages in Site Pages library"

                    # Process each page
                    foreach ($page in $pages) {
                        try {
                            # Check if _AuthorByline field exists and has value
                            if ($page._AuthorByline -and $page._AuthorByline.Email) {
                                $authorBylineInfo = [PSCustomObject]@{
                                    URL       = $TenantUrl + $page.FileRef
                                    Email     = $page._AuthorByline.Email
                                    Name      = if ($page._AuthorByline.LookupValue) { $page._AuthorByline.LookupValue } else { "" }
                                    errorcode = ""
                                }
                                $script:AuthorBylineCollection += $authorBylineInfo
                                $script:Summary.AuthorBylinesFound++
                                Write-Verbose "Found author byline: $($page._AuthorByline.Email) on page $($page.FileRef)"
                            }
                        }
                        catch {
                            Write-Warning "Failed to process page ID $($page.ID) on $siteUrl : $_"
                            $errorInfo = [PSCustomObject]@{
                                URL       = $TenantUrl + $page.FileRef
                                Email     = ""
                                Name      = ""
                                errorcode = $_.Exception.Message
                            }
                            $script:AuthorBylineCollection += $errorInfo
                            $script:Summary.Failures++
                        }
                    }
                }
                catch {
                    Write-Warning "Failed to retrieve pages from $siteUrl : $_"
                    $script:Summary.Failures++
                }
            }
        }
        catch {
            Write-Error "Failed to retrieve or process sites: $_"
            $script:Summary.Failures++
        }
    }

    end {
        # Export to CSV if requested
        if ($ExportToCsv -and $script:AuthorBylineCollection.Count -gt 0) {
            $csvPath = Join-Path -Path $OutputPath -ChildPath "AuthorbylineUsers.csv"
            
            if ($PSCmdlet.ShouldProcess($csvPath, 'Export author byline data to CSV')) {
                try {
                    $script:AuthorBylineCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8 -Delimiter "|"
                    Write-Host "`nAuthor byline data exported successfully to: $csvPath" -ForegroundColor Green
                }
                catch {
                    Write-Warning "Failed to export CSV to '$csvPath': $_"
                    $script:Summary.Failures++
                }
            }
        }
        elseif ($ExportToCsv -and $script:AuthorBylineCollection.Count -eq 0) {
            Write-Warning "No author byline data found to export to CSV"
        }

        # Display summary
        Write-Host "`n========================================" -ForegroundColor Cyan
        Write-Host "     Author Byline Export Summary" -ForegroundColor Cyan
        Write-Host "========================================" -ForegroundColor Cyan
        Write-Host "Sites Processed:       $($script:Summary.SitesProcessed)" -ForegroundColor White
        Write-Host "Pages Found:           $($script:Summary.PagesFound)" -ForegroundColor White
        Write-Host "Author Bylines Found:  $($script:Summary.AuthorBylinesFound)" -ForegroundColor Green
        Write-Host "Failures:              $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
        Write-Host "========================================`n" -ForegroundColor Cyan

        # Display results in terminal if not exporting to CSV
        if (-not $ExportToCsv -and $script:AuthorBylineCollection.Count -gt 0) {
            Write-Host "`nAuthor Byline Data:" -ForegroundColor Cyan
            $script:AuthorBylineCollection | Format-Table -AutoSize
        }
    }
}

# Example usage
Export-AuthorBylineUsers -TenantUrl "https://contoso.sharepoint.com" -ExportToCsv -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| Kasper Larsen, Fellowmind|
| Adam Wójcik (Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-author-byline-users" aria-hidden="true" />



# SharePoint Modern Page URL Report


## Summary

This report will go through all the pages in a modern site and report on the URLs within each **Quick Links** Web Part. 
This is useful for migration and new content scenarios to ensure that any placeholder or temporary links within a page are listed in a report.

![Example Screenshot](assets/example.png)

This script does have scope for the future to include other web part types e.g. Image Web Parts, Hero where user/provisioning has specified the link.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory=$true, HelpMessage="SharePoint site URL")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory=$false, HelpMessage="Path for output CSV file")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "QuickLinksReport_$timestamp.log"
    Start-Transcript -Path $transcriptPath | Out-Null

    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Cyan
    $loginOutput = m365 login --ensure 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to connect to Microsoft 365: $loginOutput"
    }

    $script:ReportCollection = @()
    $script:Summary = @{
        TotalPages = 0
        PagesWithQuickLinks = 0
        TotalQuickLinks = 0
        TotalLinks = 0
        Failures = 0
    }

    Write-Host "Starting Quick Links URL report for: $SiteUrl" -ForegroundColor Green
}

process {
    try {
        Write-Host "Retrieving all modern pages from site..." -ForegroundColor Cyan
        $pagesJson = m365 spo page list --webUrl $SiteUrl --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve pages: $pagesJson"
        }
        
        $pages = $pagesJson | ConvertFrom-Json
        $script:Summary.TotalPages = $pages.Count
        
        if ($pages.Count -eq 0) {
            Write-Host "No modern pages found in site" -ForegroundColor Yellow
            return
        }

        Write-Host "Found $($pages.Count) pages. Scanning for Quick Links web parts..." -ForegroundColor Cyan

        foreach ($page in $pages) {
            try {
                Write-Verbose "Processing page: $($page.Name)"
                
                $controlsJson = m365 spo page control list --webUrl $SiteUrl --pageName $page.Name --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve controls for page '$($page.Name)': $controlsJson"
                    $script:Summary.Failures++
                    continue
                }

                $controls = $controlsJson | ConvertFrom-Json
                $quickLinksControls = $controls | Where-Object { 
                    $_.controlData.webPartData.id -eq 'c70391ea-0b10-4ee9-b2b4-006d3fcad0cd' 
                }

                if ($quickLinksControls.Count -gt 0) {
                    $script:Summary.PagesWithQuickLinks++
                    Write-Host "  Page '$($page.Name)' - Found $($quickLinksControls.Count) Quick Links web part(s)" -ForegroundColor Yellow
                }

                foreach ($control in $quickLinksControls) {
                    $script:Summary.TotalQuickLinks++
                    $webPartTitle = $control.controlData.webPartData.title
                    $serverContent = $control.controlData.webPartData.serverProcessedContent | ConvertFrom-Json -AsHashtable
                    
                    if (-not $serverContent.links -or $serverContent.links.Count -eq 0) {
                        Write-Verbose "  Web part '$webPartTitle' has no links"
                        continue
                    }

                    $itemCount = ($serverContent.links.Keys | Where-Object { $_ -like 'items[*].sourceItem.url' }).Count

                    For($i = 0; $i -lt $itemCount; $i++) {
                        $urlPath = "items[$i].sourceItem.url"
                        $titlePath = "items[$i].title"
                        
                        $linkUrl = $serverContent.links[$urlPath]
                        $linkTitle = if ($serverContent.searchablePlainTexts.ContainsKey($titlePath)) {
                            $serverContent.searchablePlainTexts[$titlePath]
                        } else {
                            ""
                        }
                        
                        if ($linkUrl) {
                            $script:Summary.TotalLinks++
                            $script:ReportCollection += [PSCustomObject]@{
                                WebTitle = $page.Title
                                WebUrl = $SiteUrl
                                PageFileName = $page.Name
                                WebPartTitle = $webPartTitle
                                LinkTitle = $linkTitle
                                LinkUrl = $linkUrl
                            }
                            Write-Verbose "    Found link: $linkTitle -> $linkUrl"
                        }
                    }
                }
            }
            catch {
                Write-Warning "Error processing page '$($page.Name)': $_"
                $script:Summary.Failures++
                continue
            }
        }
    }
    catch {
        Write-Error "Critical error during page scan: $_"
        throw
    }
}

end {
    Stop-Transcript | Out-Null

    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Quick Links URL Report Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Total Pages Scanned      : $($script:Summary.TotalPages)" -ForegroundColor White
    Write-Host "Pages with Quick Links   : $($script:Summary.PagesWithQuickLinks)" -ForegroundColor White
    Write-Host "Total Quick Links WPs    : $($script:Summary.TotalQuickLinks)" -ForegroundColor White
    Write-Host "Total Links Extracted    : $($script:Summary.TotalLinks)" -ForegroundColor Green
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failed Pages             : $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failed Pages             : 0" -ForegroundColor Green
    }
    Write-Host "========================================" -ForegroundColor Cyan

    if ($script:ReportCollection.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "QuickLinksReport_$timestamp.csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "`nCSV Report exported to: $csvPath" -ForegroundColor Green
    } else {
        Write-Host "`nNo Quick Links found. No CSV report generated." -ForegroundColor Yellow
    }

    Write-Host "Transcript log saved to: $transcriptPath" -ForegroundColor Green
}

# Usage Examples:
#
# Example 1: Basic usage - generate report for a site
# .\Generate-QuickLinksReport.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing"
#
# Example 2: Specify custom output path
# .\Generate-QuickLinksReport.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -OutputPath "C:\Reports"
#
# Example 3: Run with verbose output
# .\Generate-QuickLinksReport.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/it" -Verbose
#
# Example 4: Preview with WhatIf (not applicable - read-only script)
# .\Generate-QuickLinksReport.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/sales"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell

# Example: .\List-UrlsInPage.ps1 -SourceSitePartUrl "SiteA" -PartTenant "contoso"
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Source e.g. Intranet-Archive")]
    [string]$SourceSitePartUrl,
    [Parameter(Mandatory = $false, HelpMessage = "Organisation Url Fragment e.g. contoso ")]
    [string]$PartTenant,
    [string]$ReportName = "URL-Report.csv"
)
begin{

    $baseUrl = "https://$($PartTenant).sharepoint.com"
    $sourceSiteUrl = "$($baseUrl)/sites/$($SourceSitePartUrl)"
    
    Write-Host "Connecting to " $sourceSiteUrl
    
    # For MFA Tenants - Interactive opens a browser window
    Connect-PnPOnline -Url $sourceSiteUrl -Interactive
    $filter = '<View>' +
                '<Query>' +
                    '<Where>' +
                        '<And>' +
                            '<Eq>'+
                                '<FieldRef Name="FSObjType" />'+
                                '<Value Type="Integer">0</Value>'+
                            '</Eq>'+
                            '<Neq>'+
                                '<FieldRef Name="_SPSitePageFlags" />'+
                                '<Value Type="Text">{Template}}</Value>'+
                            '</Neq>'+
                        '</And>' +
                    '</Where>' +
                '</Query>' +
            '</View>'

    $loc = Get-Location
    $reportPath = "$($loc)\$($ReportName)"

    '"WebTitle","WebUrl","PageFileName","WebPartTitle","LinkTitle","LinkUrl"' | Out-File $reportPath
}
process{

    Write-Host "Reading pages in site..."

    $web = Get-PnPWeb -Includes Title,Url
    $webTitle = $web.Title
    $webUrl = $web.Url
    
    $pages = Get-PnPListItem -List "SitePages" -Query $filter
            
    Foreach($page in $pages){

        $file = $page.FieldValues["FileLeafRef"]

        Write-Host " Processing Page $($file)" -ForegroundColor Cyan

        $components = Get-PnPPageComponent -Page $file
        
        # To find the Web Part ID:
        # Get-PnPPageComponent -Page "MyPage.aspx" -ListAvailable

        # You can filter based on type of web part
        # c70391ea-0b10-4ee9-b2b4-006d3fcad0cd QuickLinksWebPart
        $summaryLinks = $components | Where-Object { $_.WebPartId -eq 'c70391ea-0b10-4ee9-b2b4-006d3fcad0cd'}
        Write-Host "Found $($summaryLinks.Count) Summary Links" -ForegroundColor Yellow

        $summaryLinks | Foreach-Object{

            $wpTitle = $_.title
            Write-Host "Web Part Title: $($wpTitle)"

            $serverContent = $_.ServerProcessedContent | ConvertFrom-Json -AsHashTable
            $itemCount = $serverContent.links.Count

            #{htmlStrings, searchablePlainTexts, imageSources, links...}
            Write-Host "Item Count: $($itemCount)"
            
            For($i = 0; $i -lt $itemCount; $i++){

                $titlePath = "items[$($i)].title"
                $urlPath = "items[$($i)].sourceItem.url"

                $lnkTitle = $serverContent.searchablePlainTexts.$titlePath
                $lnkUrl = $serverContent.links.$urlPath

                if($lnkTitle -and $lnkUrl){

                    Write-Host "    Link Title: $($lnkTitle)" -ForegroundColor Cyan
                    Write-Host "    Link URL: $($lnkUrl)" -ForegroundColor Cyan
    
                    $line = '"' + $webTitle + '","' + `
                        $webUrl + '","' + `
                        $file + '","' + `
                        $wpTitle  + '","' + `
                        $lnkTitle + '","' + `
                        $lnkUrl + '"'
    
                    $line | Out-File $reportPath -Append
    
                }
            }

        }
    }
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| Paul Bullock |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-modern-page-url-report" aria-hidden="true" />

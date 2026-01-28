

# Find Web Part in Pages e.g., Twitter Web Part

## Summary

This script will find all instances of the specified Web Part on a page or (if chosen) in templates too using PnP PowerShell or CLI for Microsoft 365. The scripe example produces a report of all of the occurances of the Twitter Web Part in a site. You can specify any web part ID to find, but there is a deprecation happening soon and this maybe useful to find any occurances of the web part.


If you would like to delete the web parts, there is an existing script to do that here: [Delete Web Parts from Pages](https://pnp.github.io/script-samples/spo-remove-webpart-from-pages/README.html)


![Example Screenshot](assets/example.png)

# [PnP PowerShell](#tab/pnpps)

```powershell

<# 

Created:      Paul Bullock
Date:         04/09/2023
License:      MIT License (MIT)

.Synopsis
    Lists out use of the twitter web part in a site, as an example, although you can specify any web part ID
#>

[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Source URL e.g. https://contoso.sharepoint.com/sites/SiteA")]
    [string]$siteUrl,
    [string]$ReportName = "Twitter-WebPart-Report.csv",
    [string]$WebPartId = "f6fdf4f8-4a24-437b-a127-32e66a5dd9b4", # Twitter Web Part ID
    [switch]$IncludeTemplates
)
begin{

    Write-Host "Connecting to " $siteUrl
        
    # For MFA Tenants - Interactive opens a browser window
    $sourceConnection = Connect-PnPOnline -Url $siteUrl  -ReturnConnection -Interactive
    
    # Caml to find just the pages, not folders or templates
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

    # Caml to find pages and templates
    if($IncludeTemplates){
        $filter = '<View>' +
                    '<Query>' +
                        '<Where>' +                        
                                '<Eq>'+
                                    '<FieldRef Name="FSObjType" />'+
                                    '<Value Type="Integer">0</Value>'+
                                '</Eq>'+
                        '</Where>' +
                    '</Query>' +
                '</View>'
    }
    
    $reportPath = "$($ReportName)"
    $WebPartList = @()

}
process{

    Write-Host "Reading pages in site..."

    $web = Get-PnPWeb -Includes Title,Url
    $webTitle = $web.Title
    $webUrl = $web.Url
    
    $pages = Get-PnPListItem -List "SitePages" -Connection $sourceConnection -Query $filter
            
    Foreach($page in $pages){

        $file = $page.FieldValues["FileLeafRef"]

        Write-Host " Processing Page $($file)" -ForegroundColor Cyan

        $components = Get-PnPPageComponent -Page $file
        
        # To find the Web Part ID (type, not instance):
        # Get-PnPPageComponent -Page "MyPage.aspx" -ListAvailable

        # You can filter based on type of web part
        $webPartInstances = $components | Where-Object { $_.WebPartId -eq $WebPartId}
        Write-Host " - Found $($webPartInstances.Count) Occurrances of that web part" -ForegroundColor Yellow

        $webPartInstances | Foreach-Object{

            $wpTitle = $_.title
            Write-Host "    Web Part Title: $($wpTitle)"
            $webPartProps = $_.PropertiesJson

            $webPartLogItem = [PSCustomObject]@{

                    "WebTitle" = $webTitle
                    "WebUrl" = $webUrl
                    "PageFileName" = $page.title
                    "WebPartTitle" = $_.title
                    "WebPartProperties" = $webPartProps
                }
                                
            $WebPartList += $webPartLogItem
        }
    }

    $WebPartList | Export-Csv -Path $reportPath -NoTypeInformation

    Write-Host "Report saved to $reportPath" -ForegroundColor Green
    Write-Host "Script Complete :-)" -ForegroundColor Green
}  

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory, HelpMessage = "Source URL e.g. https://contoso.sharepoint.com/sites/SiteA")]
    [string]$SiteUrl,

    [Parameter(Mandatory, HelpMessage = "Web Part ID (GUID) to search for, e.g., f6fdf4f8-4a24-437b-a127-32e66a5dd9b4 for Twitter")]
    [string]$WebPartId,

    [Parameter(HelpMessage = "Output folder path for CSV report")]
    [string]$OutputPath = (Get-Location).Path,

    [switch]$IncludeTemplates
)

begin {
    $script:ReportCollection = @()
    $script:Summary = @{
        PagesScanned = 0
        WebPartsFound = 0
        Failures = 0
    }

    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $reportPath = Join-Path $OutputPath "WebPartReport-$timestamp.csv"
    Start-Transcript -Path (Join-Path $OutputPath "FindWebPart-$timestamp.log")

    Write-Verbose "Authenticating with Microsoft 365..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365"
    }

    Write-Host "Retrieving pages from $SiteUrl..." -ForegroundColor Cyan
    
    try {
        $pagesJson = m365 spo page list --webUrl $SiteUrl --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve pages from site"
        }
        $pages = @($pagesJson | ConvertFrom-Json)
        Write-Host "Found $($pages.Count) page(s)" -ForegroundColor Green
    }
    catch {
        throw "Error retrieving pages: $_"
    }

    if ($IncludeTemplates) {
        Write-Verbose "Retrieving page templates..."
        try {
            $templatesJson = m365 spo page template list --webUrl $SiteUrl --output json
            if ($LASTEXITCODE -eq 0) {
                $templates = @($templatesJson | ConvertFrom-Json)
                $pages += $templates
                Write-Host "Found $($templates.Count) template(s)" -ForegroundColor Green
            }
        }
        catch {
            Write-Warning "Failed to retrieve templates: $_"
        }
    }

    $script:AllPages = $pages
}

process {
    foreach ($page in $script:AllPages) {
        $pageName = $page.Name
        Write-Verbose "Processing page: $pageName"
        
        try {
            $controlsJson = m365 spo page control list --webUrl $SiteUrl --pageName $pageName --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve controls from $pageName"
                $script:Summary.Failures++
                continue
            }

            $controls = @($controlsJson | ConvertFrom-Json)
            
            $matchingControls = $controls | Where-Object { $_.controlData.webPartId -eq $WebPartId }
            
            if ($matchingControls.Count -gt 0) {
                Write-Host "  Found $($matchingControls.Count) matching web part(s) in $pageName" -ForegroundColor Yellow
                
                foreach ($control in $matchingControls) {
                    $propertiesJson = if ($control.controlData.webPartData.properties) {
                        ($control.controlData.webPartData.properties | ConvertTo-Json -Depth 10 -Compress)
                    } else {
                        ""
                    }

                    $reportItem = [PSCustomObject]@{
                        PageTitle = $page.Title
                        PageUrl = $page.AbsoluteUrl
                        WebPartTitle = $control.title
                        WebPartId = $WebPartId
                        WebPartProperties = $propertiesJson
                    }

                    $script:ReportCollection += $reportItem
                    $script:Summary.WebPartsFound++
                }
            }

            $script:Summary.PagesScanned++
        }
        catch {
            Write-Warning "Error processing $pageName: $_"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Host "`nExporting report to $reportPath..." -ForegroundColor Cyan
    $script:ReportCollection | Export-Csv -Path $reportPath -NoTypeInformation

    Write-Host "`nSummary:" -ForegroundColor Cyan
    Write-Host "  Pages scanned: $($script:Summary.PagesScanned)" -ForegroundColor Green
    Write-Host "  Web parts found: $($script:Summary.WebPartsFound)" -ForegroundColor Green
    if ($script:Summary.Failures -gt 0) {
        Write-Host "  Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
    Write-Host "`nReport saved to: $reportPath" -ForegroundColor Green
    
    Stop-Transcript
}

# Example 1: Find Twitter web part in all pages
# .\Find-WebPartInPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -WebPartId "f6fdf4f8-4a24-437b-a127-32e66a5dd9b4"

# Example 2: Find web part including templates with verbose output
# .\Find-WebPartInPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -WebPartId "f6fdf4f8-4a24-437b-a127-32e66a5dd9b4" -IncludeTemplates -Verbose

# Example 3: Find Bing Maps web part with custom output path
# .\Find-WebPartInPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -WebPartId "e377ea37-9047-43b9-8cdb-a761be2f8e09" -OutputPath "C:\Reports"

# Example 4: Find any custom SPFx web part by its ID
# .\Find-WebPartInPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/intranet" -WebPartId "a5df8fdf-b508-4b91-aed9-46d46f798c68"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Paul Bullock |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-find-web-part-in-pages" aria-hidden="true" />

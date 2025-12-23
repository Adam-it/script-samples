

# Export HTML content from SharePoint Online modern pages

## Summary

This sample script exports all SharePoint Online modern pages as .html files, focusing on text content. This is useful when analysing the actual text with tools that don't support SharePoint Online but support HTML. This sample is available in both PnP PowerShell and CLI for Microsoft 365.

Files, images and videos included in the pages are not exported.

![Example Screenshot](assets/example.png)

# [PnP PowerShell](#tab/pnpps)

```powershell
$url = "<spo site url>"
$destFolder = "C:\SitePages"

$ErrorActionPreference = 'Stop'

# Create the destination folder if it doesn't exist
mkdir $destFolder -ErrorAction:SilentlyContinue | Out-Null

# Connect to SPO using PnP PowerShell
Connect-PnPOnline $url -Interactive
# Get all the pages. Credits to https://pnp.github.io/script-samples/spo-export-stream-classic-webparts/README.html to filter only pages
$list = Get-PnPList "SitePages"
$pageItems = Get-PnPListItem -List $list -Fields CanvasContent1,Title,FileLeafRef | Where-Object { $_["FileLeafRef"] -like "*.aspx" }
foreach ($pageItem in $pageItems)
{
    try
    {
        # Save the html content of each page to a .html file
        $content = $pageItem["CanvasContent1"]
        $filename = $pageItem["FileLeafRef"]
        # Additional metadata could be added here in its own paragraph
        $prefix = "<div><h1>$($pageItem["Title"])</h1></div>"
        $prefix + $content | Out-File -LiteralPath "$($destFolder)\$($filename.Replace(".aspx",".html"))"
    }
    catch
    {
        Write-Host $_
    }
}
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory, HelpMessage = "SharePoint site URL")]
    [string]$SiteUrl,
    
    [Parameter(HelpMessage = "Output directory path for exported HTML files")]
    [string]$OutputPath = ".",
    
    [Parameter(HelpMessage = "Include page title as H1 heading in exported HTML")]
    [switch]$IncludeTitle
)

begin {
    Write-Verbose "Verifying CLI for Microsoft 365 login status..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to verify login status. Please run 'm365 login' first."
    }
    
    if (-not (Test-Path $OutputPath)) {
        throw "Output path does not exist: $OutputPath"
    }
    
    $script:Summary = @{
        TotalPages = 0
        ExportedPages = 0
        FailedPages = 0
    }
}

process {
    Write-Verbose "Retrieving modern pages from: $SiteUrl"
    $pagesJson = m365 spo page list --webUrl $SiteUrl --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve pages: $pagesJson"
    }
    
    $pages = @($pagesJson | ConvertFrom-Json)
    $script:Summary.TotalPages = $pages.Count
    Write-Verbose "Found $($pages.Count) pages to export"
    
    foreach ($page in $pages) {
        $pageName = $page.Name
        Write-Verbose "Processing: $pageName"
        
        if ($PSCmdlet.ShouldProcess($pageName, "Export page to HTML")) {
            try {
                $pageDetailsJson = m365 spo page get --webUrl $SiteUrl --name $pageName --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    throw $pageDetailsJson
                }
                
                $pageDetails = $pageDetailsJson | ConvertFrom-Json
                $htmlContent = $pageDetails.CanvasContent1
                
                $outputFileName = $pageName -replace '\.aspx$', '.html'
                $outputFilePath = Join-Path $OutputPath $outputFileName
                
                if ($IncludeTitle) {
                    $pageTitle = $pageDetails.Title
                    $prefix = "<div><h1>$pageTitle</h1></div>"
                    $finalContent = $prefix + $htmlContent
                } else {
                    $finalContent = $htmlContent
                }
                
                $finalContent | Out-File -FilePath $outputFilePath -Encoding UTF8
                $script:Summary.ExportedPages++
            }
            catch {
                Write-Warning "Failed to export '$pageName': $_"
                $script:Summary.FailedPages++
            }
        }
    }
}

end {
    Write-Host "`nSummary:" -ForegroundColor Cyan
    Write-Host "  Total pages found: $($script:Summary.TotalPages)" -ForegroundColor White
    
    $exportedColor = if ($script:Summary.ExportedPages -gt 0) { "Green" } else { "White" }
    Write-Host "  Pages exported: $($script:Summary.ExportedPages)" -ForegroundColor $exportedColor
    
    if ($script:Summary.FailedPages -gt 0) {
        Write-Host "  Failed exports: $($script:Summary.FailedPages)" -ForegroundColor Red
    }
    
    Write-Host "`nHTML files saved to: $OutputPath" -ForegroundColor Green
}

# Example usage:
# .\\Export-PagesToHtml.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing"
# .\\Export-PagesToHtml.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -OutputPath "C:\\ExportedPages"
# .\\Export-PagesToHtml.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -IncludeTitle -Verbose
# .\\Export-PagesToHtml.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -WhatIf
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

This sample reuses parts of [https://pnp.github.io/script-samples/spo-export-stream-classic-webparts/README.html](https://pnp.github.io/script-samples/spo-export-stream-classic-webparts/README.html)

## Contributors

| Author(s) |
|-----------|
| Giacomo Pozzoni |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it)|


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-page-html" aria-hidden="true" />

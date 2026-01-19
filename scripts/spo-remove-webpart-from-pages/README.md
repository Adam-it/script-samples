

# Remove modern web parts from pages

## Summary

Script will remove web part(s) on multiple pages by their Title. You can optionally filter to specific pages by name.

## Implementation

- Open Windows PowerShell ISE
- Edit Script and add details like SharePoint tenant URL, Term groups, and the output directory
- Press run

[!INCLUDE [Delete Warning](../../docfx/includes/DELETE-WARN.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell

Function Remove-WebpartFromPages() {
    PARAM (
        [Parameter(Mandatory = $true)]
        [string]$SiteURL,

        [Parameter(Mandatory = $false)]
        [string[]]$WebPartIds,

        [Parameter(Mandatory = $false)]
        [string]$ContentTypeId,

        [Parameter(Mandatory = $false)]
        [string[]]$Pages
    )

    Try {
            ## Connect to SharePoint Online site  
            Write-Host "Connect to $($SiteURL)"
            Connect-PnPOnline -URL $SiteURL -UseWebLogin

            $pageItems = @()
            $skippedPages = @()

            # If page parameter is empty, loop through all pages
            if ($Pages.Length -lt 1) {
                $pageItems = Get-PnPListItem -List "Site Pages"
                $Pages = $pageItems | ForEach-Object { $_["FileLeafRef"] }
            }

            if($Pages.Length -lt 1){
                $pageItems = Get-PnPListItem -List "Site Pages"
            }
            
            if ($ContentTypeId) {
                $pageItems = $pageItems | Where-Object {$_["ContentTypeId"].toString() -eq $ContentTypeId}
            }
            
            if($pageItems.Length -ge 1){
                $Pages = $pageItems | ForEach-Object{ $_["FileLeafRef"] }
            }
            
            $Pages | ForEach-Object {
                $fileLeafRef = $_
                Write-Host "Processing $fileLeafRef"
                try {
                    $page = Get-PnPPage -Identity $fileLeafRef
                    if($WebPartIds.Length -ge 1){
                        $controls = $page.Controls | Where-Object {$WebPartIds -contains $_.Title -or $WebPartIds -contains $_.WebPartId}
                    }                    
                    $controls | ForEach-Object {
                        Write-Host "Removing web part: $($_.Title)"
                        Remove-PnPPageComponent -Page $page -InstanceId $_.InstanceId -Force
                        Write-Host "Web part $($_.Title) removed successfully from $($fileLeafRef)" -ForegroundColor "green"
                    }
                }
                catch {
                    $skippedPages += $fileLeafRef                    
                    Write-Host "Skipped $fileLeafRef" -ForegroundColor "yellow"
                }
            
            }
        }

        Catch {
            Write-Host "Error: $($_.Exception)" -ForegroundColor Red
            Break
        }

        ## Disconnect the context  
        Disconnect-PnPOnline  
}

Remove-WebPartFromPages -SiteURL https://contoso.sharepoint.com -WebPartIds "News","0ec51ebc-4754-4ef4-a953-1a3adb4b4d8c" -Pages "home.aspx","pnp.aspx"

# More examples
<#
# Specific content type
Remove-WebPartFromPages -SiteURL https://contoso.sharepoint.com -WebPartIds "News" -ContentTypeId "0x0101009D1CB255DA764"

#>


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Web part titles to remove (case-insensitive partial match)")]
    [string[]]$WebPartTitles,

    [Parameter(Mandatory = $false, HelpMessage = "Specific page names to process")]
    [string[]]$PageNames,

    [Parameter(Mandatory = $false, HelpMessage = "Output folder for CSV report and transcript")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath)) {
            throw "Output path does not exist: $OutputPath"
        }
    }

    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365"
    }

    $script:Summary = @{
        PagesProcessed   = 0
        ControlsRemoved  = 0
        PagesSkipped     = 0
        Failures         = 0
    }

    $script:ReportCollection = @()

    $timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
    Start-Transcript -Path "$OutputPath\RemoveWebParts_$timestamp.log"
}

process {
    Write-Host "
Retrieving pages from site..." -ForegroundColor Cyan
    $pagesResult = m365 spo page list --webUrl $SiteUrl --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve pages: $pagesResult"
    }

    $pages = @($pagesResult | ConvertFrom-Json)
    Write-Host "Found $($pages.Count) total pages" -ForegroundColor White

    if ($PageNames -and $PageNames.Count -gt 0) {
        Write-Host "Filtering by page names: $($PageNames -join ', ')" -ForegroundColor White
        $pages = $pages | Where-Object { $PageNames -contains $_.Name }
        Write-Host "$($pages.Count) pages match name filter" -ForegroundColor White
    }

    if ($pages.Count -eq 0) {
        Write-Warning "No pages found matching filters"
        return
    }

    Write-Host "Processing $($pages.Count) pages..." -ForegroundColor Cyan

    foreach ($page in $pages) {
        Write-Host "Processing page: $($page.Name)" -ForegroundColor White
        
        try {
            $controlsResult = m365 spo page control list --webUrl $SiteUrl --pageName $page.Name --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "  Failed to retrieve controls: $controlsResult"
                $script:Summary.PagesSkipped++
                continue
            }

            $allControls = @($controlsResult | ConvertFrom-Json)
            $matchingControls = @()

            foreach ($control in $allControls) {
                if ($WebPartTitles | Where-Object { $control.title -like "*$_*" }) {
                    $matchingControls += $control
                }
            }

            if ($matchingControls.Count -eq 0) {
                Write-Host "  No matching web parts found" -ForegroundColor Gray
                $script:Summary.PagesSkipped++
                continue
            }

            Write-Host "  Found $($matchingControls.Count) matching web parts" -ForegroundColor White

            foreach ($control in $matchingControls) {
                $target = "$($page.Name) - '$($control.title)'"
                $action = "Remove web part"

                if ($PSCmdlet.ShouldProcess($target, $action)) {
                    try {
                        m365 spo page control remove --webUrl $SiteUrl --pageName $page.Name --id $control.id --force 2>&1 | Out-Null
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "    Failed to remove '$($control.title)'"
                            $script:Summary.Failures++
                            
                            $script:ReportCollection += [PSCustomObject]@{
                                PageName       = $page.Name
                                ControlId      = $control.id
                                WebPartId      = $control.controlData.webPartId
                                WebPartTitle   = $control.title
                                Status         = "Failed"
                                ErrorMessage   = "CLI returned non-zero exit code"
                            }
                        }
                        else {
                            Write-Host "    SUCCESS: Removed '$($control.title)'" -ForegroundColor Green
                            $script:Summary.ControlsRemoved++
                            
                            $script:ReportCollection += [PSCustomObject]@{
                                PageName       = $page.Name
                                ControlId      = $control.id
                                WebPartId      = $control.controlData.webPartId
                                WebPartTitle   = $control.title
                                Status         = "Success"
                                ErrorMessage   = ""
                            }
                        }
                    }
                    catch {
                        Write-Warning "    Failed: $($_.Exception.Message)"
                        $script:Summary.Failures++
                        
                        $script:ReportCollection += [PSCustomObject]@{
                            PageName       = $page.Name
                            ControlId      = $control.id
                            WebPartId      = $control.controlData.webPartId
                            WebPartTitle   = $control.title
                            Status         = "Failed"
                            ErrorMessage   = $_.Exception.Message
                        }
                    }
                }
                else {
                    Write-Host "    WHATIF: Would remove '$($control.title)'" -ForegroundColor Yellow
                    $script:Summary.ControlsRemoved++
                    
                    $script:ReportCollection += [PSCustomObject]@{
                        PageName       = $page.Name
                        ControlId      = $control.id
                        WebPartId      = $control.controlData.webPartId
                        WebPartTitle   = $control.title
                        Status         = "WhatIf"
                        ErrorMessage   = ""
                    }
                }
            }

            $script:Summary.PagesProcessed++
        }
        catch {
            Write-Warning "  Error: $($_.Exception.Message)"
            $script:Summary.PagesSkipped++
            continue
        }
    }
}

end {
    Stop-Transcript

    $timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
    $csvPath = "$OutputPath\RemovedWebParts_$timestamp.csv"
    $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation
    Write-Host "CSV report exported to: $csvPath" -ForegroundColor Cyan

    Write-Host "=== Summary ===" -ForegroundColor Cyan
    Write-Host "Pages Processed   : $($script:Summary.PagesProcessed)" -ForegroundColor White
    Write-Host "Controls Removed  : $($script:Summary.ControlsRemoved)" -ForegroundColor Green
    Write-Host "Pages Skipped     : $($script:Summary.PagesSkipped)" -ForegroundColor Yellow
    $failColor = if ($script:Summary.Failures -eq 0) { "Green" } else { "Red" }
    Write-Host "Failures          : $($script:Summary.Failures)" -ForegroundColor $failColor
}

# Remove web parts by title from all pages with WhatIf
# .\Remove-WebpartFromPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -WebPartTitles "News", "Events" -WhatIf

# Remove web parts by title from specific pages
# .\Remove-WebpartFromPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -WebPartTitles "News" -PageNames "Home.aspx", "About.aspx"

# Remove from all pages in site
# .\Remove-WebpartFromPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -WebPartTitles "Quick Links", "Hero"

# Remove with verbose output
# .\Remove-WebpartFromPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -WebPartTitles "News" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Ramin Ahmadi](https://github.com/ahmadiramin) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-remove-webpart-from-pages" aria-hidden="true" />

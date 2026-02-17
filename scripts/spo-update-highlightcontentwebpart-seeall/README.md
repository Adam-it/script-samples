

# Hide the 'See All' Button in the Highlighted Content Web Part

## Summary

Recently, I encountered an issue with the "Show title and commands" toggle in the out-of-the-box Highlighted Content web part. It stopped working on both my development and customer tenant.

![ToggleOff](assets/HighlightWebPart.png)

While awaiting a resolution from Microsoft, I decided to find a workaround. The solution involves updating the **PropertiesJson** property of the web part. This sample is available in both PnP PowerShell and CLI for Microsoft 365 v11.4.0+.
 
# [PnP PowerShell](#tab/pnpps)

```PowerShell

# Connect to your SharePoint site
Connect-PnPOnline -Url "https://contoso.sharepoint.com/sites/Project" -Interactive

# Specify the page URL
$pageUrl = "ProjectHome.aspx"

# Get the page and its web parts
$page = Get-PnPClientSidePage -Identity $pageUrl
$webParts = $page.Controls | Where-Object { $_.Title -eq 'Highlighted content' } 

# Loop through each web part
foreach ($webPart in $webParts) {
        # Update isTitleEnabled property within PropertiesJson
        $jsonUp = $webpart.PropertiesJson.Replace('"isTitleEnabled":true','"isTitleEnabled":false') 
        
        Set-PnPPageWebPart -Page $pageUrl -Identity $webPart.InstanceId -PropertiesJson $jsonUp
    }

# Disconnect from the SharePoint site
Disconnect-PnPOnline

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage="SharePoint site URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory, HelpMessage="Page name (e.g., ProjectHome.aspx)")]
    [string]$PageName,
    
    [Parameter(HelpMessage="Output path for transcript")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Host "Hiding 'See All' button in Highlighted Content web parts..." -ForegroundColor Cyan
    Write-Host "Site: $SiteUrl" -ForegroundColor Cyan
    Write-Host "Page: $PageName`n" -ForegroundColor Cyan

    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365"
    }

    $pageJson = m365 spo page get --name $PageName --webUrl $SiteUrl --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Page '$PageName' not found at site '$SiteUrl'"
    }

    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath)) {
            throw "Invalid OutputPath: $OutputPath"
        }
    }

    $script:Summary = @{
        WebPartsFound = 0
        WebPartsUpdated = 0
        AlreadyHidden = 0
        Failures = 0
    }

    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = Join-Path $OutputPath "HideHighlightedContentSeeAll_$timestamp.log"
    Start-Transcript -Path $transcriptPath
}

process {
    Write-Verbose "Retrieving all controls on page '$PageName'..."
    $controlsJson = m365 spo page control list --pageName $PageName --webUrl $SiteUrl --output json
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve page controls"
    }

    $controls = @($controlsJson | ConvertFrom-Json)
    $highlightedContentWebParts = $controls | Where-Object { $_.title -eq 'Highlighted content' }
    $script:Summary.WebPartsFound = $highlightedContentWebParts.Count

    if ($highlightedContentWebParts.Count -eq 0) {
        Write-Host "No Highlighted Content web parts found on this page." -ForegroundColor Yellow
        return
    }

    Write-Host "Found $($highlightedContentWebParts.Count) Highlighted Content web part(s)`n" -ForegroundColor Green

    foreach ($webPart in $highlightedContentWebParts) {
        try {
            Write-Verbose "Processing web part ID: $($webPart.id)"

            $webPartJson = m365 spo page control get --id $webPart.id --pageName $PageName --webUrl $SiteUrl --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to get properties for web part ID: $($webPart.id)"
                $script:Summary.Failures++
                continue
            }

            $webPartData = $webPartJson | ConvertFrom-Json
            $propertiesJson = $webPartData.webPartData

            if ($propertiesJson -notlike '*"isTitleEnabled":true*') {
                Write-Host "Web part ID $($webPart.id) already has 'See All' button hidden. Skipping." -ForegroundColor Gray
                $script:Summary.AlreadyHidden++
                continue
            }

            $updatedJson = $propertiesJson.Replace('"isTitleEnabled":true', '"isTitleEnabled":false')

            if ($PSCmdlet.ShouldProcess("Web part ID: $($webPart.id)", "Hide 'See All' button")) {
                m365 spo page control set --id $webPart.id --pageName $PageName --webUrl $SiteUrl --webPartData $updatedJson

                if ($LASTEXITCODE -eq 0) {
                    Write-Host "Successfully updated web part ID: $($webPart.id)" -ForegroundColor Green
                    $script:Summary.WebPartsUpdated++
                }
                else {
                    Write-Warning "Failed to update web part ID: $($webPart.id)"
                    $script:Summary.Failures++
                }
            }
        }
        catch {
            Write-Warning "Error processing web part ID $($webPart.id): $($_.Exception.Message)"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Host "`n=== Update Summary ===" -ForegroundColor Cyan
    Write-Host "Web parts found: $($script:Summary.WebPartsFound)" -ForegroundColor White
    Write-Host "Web parts updated: $($script:Summary.WebPartsUpdated)" -ForegroundColor White
    Write-Host "Already hidden: $($script:Summary.AlreadyHidden)" -ForegroundColor White

    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
    else {
        Write-Host "Failures: 0" -ForegroundColor Green
    }

    Stop-Transcript
}

# Usage examples:
# .\Hide-HighlightedContentSeeAll.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Project" -PageName "ProjectHome.aspx"

# .\Hide-HighlightedContentSeeAll.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Project" -PageName "ProjectHome.aspx" -WhatIf

# .\Hide-HighlightedContentSeeAll.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Project" -PageName "ProjectHome.aspx" -Verbose

# .\Hide-HighlightedContentSeeAll.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Project" -PageName "ProjectHome.aspx" -OutputPath "C:\\Reports"

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

## Source Credit

Sample first appeared on [How to Hide the 'See All' Button in the Highlighted Content Web Part using PnP PowerShell](https://reshmeeauckloo.com/posts/powershell_highlightwebpart_hideseeall/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011)|
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]

<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-highlightcontentwebpart-seeall" aria-hidden="true" />

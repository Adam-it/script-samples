

# Copy Webparts From One Page To Another Page

## Summary

This sample read Site Url, Source page and destination page from user and then we will copy webparts from source page to destination page

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as below,
- First, we will read site URL from user and connect to Site.
    - Then we will read source page and destination page from user.
    - And then we will get all the webparts from source page.
    - If webparts will be found then we will add these webparts to destination page.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Site URL where the pages are located")]
    [ValidatePattern('^https://')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Source page name with web parts to copy (e.g., Home.aspx)")]
    [ValidateNotNullOrEmpty()]
    [string]$SourcePageName,

    [Parameter(Mandatory = $true, HelpMessage = "Destination page name where web parts will be copied (e.g., NewHome.aspx)")]
    [ValidateNotNullOrEmpty()]
    [string]$DestinationPageName
)

begin {
    $logFile = ".\copy-webparts-$(Get-Date -Format 'yyyyMMddHHmmss').log"
    Start-Transcript -Path $logFile
    Write-Verbose "Transcript logging to: $logFile"

    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365."
    }

    Write-Verbose "Retrieving web parts from source page: $SourcePageName"
    $sourceControlsJson = m365 spo page control list --webUrl $SiteUrl --pageName $SourcePageName --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve controls from source page '$SourcePageName'. Error: $sourceControlsJson"
    }

    $script:SourceControls = @($sourceControlsJson | ConvertFrom-Json)
    Write-Verbose "Found $($script:SourceControls.Count) web part(s) on source page."

    if ($script:SourceControls.Count -eq 0) {
        Write-Host "No web parts found on source page '$SourcePageName'. Nothing to copy." -ForegroundColor Yellow
        Stop-Transcript
        return
    }

    $script:Summary = [PSCustomObject]@{
        WebPartsCopied = 0
        Failed         = 0
    }
}

process {
    $totalControls = $script:SourceControls.Count
    $currentControl = 0

    foreach ($control in $script:SourceControls) {
        $currentControl++
        $controlTitle = if ($control.title) { $control.title } else { "Untitled" }
        
        Write-Progress -Activity "Copying web parts" `
                       -Status "Processing '$controlTitle' ($currentControl of $totalControls)" `
                       -PercentComplete (($currentControl / $totalControls) * 100)

        Write-Verbose "Processing web part: $controlTitle (ID: $($control.id))"

        try {
            Write-Verbose "  Retrieving full web part details..."
            $controlDetailsJson = m365 spo page control get --webUrl $SiteUrl --pageName $SourcePageName --id $control.id --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve details for web part '$controlTitle'. Error: $controlDetailsJson"
                $script:Summary.Failed++
                continue
            }

            $controlDetails = $controlDetailsJson | ConvertFrom-Json
            $webPartProperties = $controlDetails.webPartData.properties | ConvertTo-Json -Depth 100 -Compress

            $sectionIndex = $control.controlData.position.sectionIndex
            $columnIndex = $control.controlData.position.columnIndex
            $controlIndex = $control.controlData.position.controlIndex
            $zoneIndex = $control.controlData.position.zoneIndex

            Write-Verbose "  Position: Section=$sectionIndex, Column=$columnIndex, Order=$controlIndex, Zone=$zoneIndex"

            if ($PSCmdlet.ShouldProcess("$DestinationPageName - Add '$controlTitle' at section $sectionIndex", 'Add web part')) {
                Write-Verbose "  Adding web part to destination page..."
                
                $addArgs = @(
                    'spo', 'page', 'clientsidewebpart', 'add',
                   '--webUrl', $SiteUrl,
                   '--pageName', $DestinationPageName,
                    '--webPartId', $control.controlData.webPartId,
                   '--webPartProperties', $webPartProperties
               )

                if ($zoneIndex -eq 2) {
                    $addArgs += '--verticalSection'
                } else {
                    $addArgs += @('--section', $sectionIndex, '--column', $columnIndex)
                }

                $addArgs += @('--order', $controlIndex)

                $addResult = m365 @addArgs 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to add web part '$controlTitle' to destination page. Error: $addResult"
                    $script:Summary.Failed++
                    continue
                }

                Write-Verbose "  Successfully added '$controlTitle'."
                $script:Summary.WebPartsCopied++
            }
        }
        catch {
            Write-Warning "Unexpected error processing web part '$controlTitle': $_"
            $script:Summary.Failed++
            continue
        }
    }

   Write-Progress -Activity "Copying web parts" -Completed

    if ($script:Summary.WebPartsCopied -gt 0) {
        if ($PSCmdlet.ShouldProcess($DestinationPageName, 'Publish page')) {
            Write-Verbose "Publishing destination page: $DestinationPageName"
            $publishResult = m365 spo page publish --webUrl $SiteUrl --name $DestinationPageName 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Web parts copied but failed to publish destination page. Error: $publishResult"
            } else {
                Write-Verbose "Destination page published successfully."
            }
        }
    }
}

end {
    Write-Host "\n========================================" -ForegroundColor Cyan
    Write-Host "Web Parts Copy Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Web parts copied: " -NoNewline -ForegroundColor Green
    Write-Host $script:Summary.WebPartsCopied -ForegroundColor White
    Write-Host "Failed:           " -NoNewline -ForegroundColor $(if ($script:Summary.Failed -gt 0) { 'Red' } else { 'Green' })
    Write-Host $script:Summary.Failed -ForegroundColor White
    Write-Host "========================================\n" -ForegroundColor Cyan

    Stop-Transcript
}

<#
# Usage Examples:

# Example 1: Copy all web parts from Home.aspx to NewHome.aspx
.\Copy-WebPartsToPage.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" `
    -SourcePageName "Home.aspx" `
    -DestinationPageName "NewHome.aspx"

# Example 2: Copy with WhatIf to preview changes
.\Copy-WebPartsToPage.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" `
    -SourcePageName "Home.aspx" `
    -DestinationPageName "Template.aspx" `
    -WhatIf

# Example 3: Verbose output for troubleshooting
.\Copy-WebPartsToPage.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" `
    -SourcePageName "Home.aspx" `
    -DestinationPageName "NewHome.aspx" `
    -Verbose

# Example 4: Copy web parts from template to multiple pages (pipeline)
@("Page1.aspx", "Page2.aspx", "Page3.aspx") | ForEach-Object {
    .\Copy-WebPartsToPage.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" `
        -SourcePageName "Template.aspx" `
        -DestinationPageName $_
}
#>

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell

$username = "user@domain.onmicrosoft.com"
$password = "********"
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$Creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd

#Login to SharePoint Site
Function ConnectToSPSite() {
    try {
        $SourceSiteUrl = Read-Host "Please enter source Site URL"
        if ($SourceSiteUrl) {
            Write-Host "Connecting to Site :'$($SourceSiteUrl)'..." -ForegroundColor Yellow  
            Connect-PnPOnline -Url $SourceSiteUrl -Credentials $Creds
            Write-Host "Connection Successfull to site: '$($SourceSiteUrl)'" -ForegroundColor Green              
            GetWebparts
        }
        else {
            Write-Host "Source Site URL is empty" -ForegroundColor Red
        }
    }
    catch {
        Write-Host "Error in connecting to Site:'$($SiteUrl)'" $_.Exception.Message -ForegroundColor Red               
    } 
}

Function GetWebparts {
    try {        
        $PageName = Read-Host "Please enter page name from where you want to copy webparts like 'Home.aspx'"
        if ($PageName) {
            Write-Host "Getting webparts from source page" -ForegroundColor Yellow  
            $page = Get-PnPClientSidePage -Identity $PageName          
            $webParts = $page.Controls  
            $WebpartsCount = $page.Controls.Count
            Write-Host "Found no. of webparts: " $WebpartsCount -ForegroundColor Gray  
            $DestinationPage = Read-Host "Please enter page name where you want to copy webparts like 'Home'"
            if ($WebpartsCount -gt 0) {
                Write-Host "Adding webparts to the page: " $DestinationPage -ForegroundColor Yellow  
                foreach ($wp in $webParts) {
                    try {                        
                        Add-PnPClientSideWebPart -Page $DestinationPage -Component $wp.Title -WebPartProperties $wp.PropertiesJson -Section $wp.Section.Order -Column $wp.Column.LayoutIndex -Order $wp.Order
                    }
                    catch {
                        Write-Host "Error in adding webparts'" $_.Exception.Message -ForegroundColor Red               
                    }
                }
                Write-Host "Added all the webparts" -ForegroundColor Green  
            }
            else {
                Write-Host "No webparts found'"-ForegroundColor Gray               
            }
        }
        else {
            Write-Host "Page name is empty" -ForegroundColor Red
        }
    }
    catch {
        Write-Host "Error in getting webparts from:'$($PageName)'" $_.Exception.Message -ForegroundColor Red               
    } 
}

Function StartProcessing { 
    ConnectToSPSite 
}

StartProcessing


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Chandani Prajapati |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-copy-webparts-to-another-page" aria-hidden="true" />

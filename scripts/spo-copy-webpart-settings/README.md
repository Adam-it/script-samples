

# Copy custom SPFx web part settings from one page to other pages

## Summary

Say we have lots of pages with a custom SPFx web part in the same location (section, column, order). 

If that web part needs to updated on all such pages then we can use this script.

We simply need to update the web part in one page. 

After that we can use this script to copy those updates on to other pages.

While using this script, we input the link of the page where the web part was updated (this page acts like the template page). 

We then specify the section, column and order of where the web part is.

After that we specify the section, column and order of where the web part is on the destination pages i.e. the pages where the web part needs to be updated.

![Example Screenshot](assets/example.png)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "The site URL where the pages are located")]
    [ValidatePattern('^https://')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "The ID of the SPFx web part (from manifest)")]
    [ValidateNotNullOrEmpty()]
    [string]$WebPartId,

    [Parameter(Mandatory = $true, HelpMessage = "The name of the source page with updated settings (e.g., page-1.aspx)")]
    [ValidateNotNullOrEmpty()]
    [string]$SourcePageName,

    [Parameter(Mandatory = $false, HelpMessage = "Section number on source page (0-based)")]
    [int]$SourceSection = 0,

    [Parameter(Mandatory = $false, HelpMessage = "Order/control index within section on source page (0-based)")]
    [int]$SourceOrder = 0,

    [Parameter(Mandatory = $true, HelpMessage = "Destination page names where settings should be copied (e.g., page-2.aspx, page-3.aspx)")]
    [ValidateNotNullOrEmpty()]
    [string[]]$DestinationPageNames,

    [Parameter(Mandatory = $false, HelpMessage = "Section number on destination pages (0-based)")]
    [int]$DestinationSection = 0,

    [Parameter(Mandatory = $false, HelpMessage = "Order/control index within section on destination pages (0-based)")]
    [int]$DestinationOrder = 0
)

begin {
    $logFile = ".\copy-webpart-settings-$(Get-Date -Format 'yyyyMMddHHmmss').log"
    Start-Transcript -Path $logFile
    Write-Verbose "Transcript logging to: $logFile"

    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365."
    }

    Write-Verbose "Retrieving web part settings from source page: $SourcePageName"
    $sourceControlsJson = m365 spo page control list --webUrl $SiteUrl --pageName $SourcePageName --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve controls from source page '$SourcePageName'. Error: $sourceControlsJson"
    }

    $sourceControls = @($sourceControlsJson | ConvertFrom-Json)
    Write-Verbose "Found $($sourceControls.Count) control(s) on source page."

    # CLI uses 1-based indexing for sectionIndex and controlIndex (order within section)
    # Convert from 0-based params to 1-based for filtering
    $targetSectionIndex = $SourceSection + 1
    $targetControlIndex = $SourceOrder + 1

    $sourceControl = $sourceControls | Where-Object {
        $_.id -eq $WebPartId -and
        $_.controlData.position.sectionIndex -eq $targetSectionIndex -and
        $_.controlData.position.controlIndex -eq $targetControlIndex
    }

    if (-not $sourceControl) {
        throw "Web part with ID '$WebPartId' not found at section $SourceSection, order $SourceOrder on source page '$SourcePageName'."
    }

    Write-Verbose "Found source web part. Extracting properties..."
    $sourceControlId = $sourceControl.id

    $sourceControlDetailsJson = m365 spo page control get --webUrl $SiteUrl --pageName $SourcePageName --id $sourceControlId --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve source control details. Error: $sourceControlDetailsJson"
    }

    $sourceControlDetails = $sourceControlDetailsJson | ConvertFrom-Json
    $sourceWebPartProperties = $sourceControlDetails.webPartData.properties | ConvertTo-Json -Depth 100 -Compress

    Write-Verbose "Source web part properties extracted successfully."

    $script:Summary = [PSCustomObject]@{
        Saved   = 0
        Skipped = 0
    }
}

process {
    foreach ($destinationPageName in $DestinationPageNames) {
        Write-Verbose "Processing destination page: $destinationPageName"

        try {
            $destControlsJson = m365 spo page control list --webUrl $SiteUrl --pageName $destinationPageName --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Skipped '$destinationPageName': Failed to retrieve controls. Error: $destControlsJson"
                $script:Summary.Skipped++
                continue
            }

            $destControls = @($destControlsJson | ConvertFrom-Json)

            $destTargetSectionIndex = $DestinationSection + 1
            $destTargetControlIndex = $DestinationOrder + 1

            $destControl = $destControls | Where-Object {
                $_.id -eq $WebPartId -and
                $_.controlData.position.sectionIndex -eq $destTargetSectionIndex -and
                $_.controlData.position.controlIndex -eq $destTargetControlIndex
            }

            if (-not $destControl) {
                Write-Warning "Skipped '$destinationPageName': Web part with ID '$WebPartId' not found at section $DestinationSection, order $DestinationOrder."
                $script:Summary.Skipped++
                continue
            }

            if ($PSCmdlet.ShouldProcess($destinationPageName, 'Update web part properties')) {
                Write-Verbose "  Updating web part properties on '$destinationPageName'..."
                $updateResult = m365 spo page control set --webUrl $SiteUrl --pageName $destinationPageName --id $destControl.id --webPartProperties $sourceWebPartProperties 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Skipped '$destinationPageName': Failed to update web part properties. Error: $updateResult"
                    $script:Summary.Skipped++
                    continue
                }

                Write-Verbose "  Publishing page '$destinationPageName'..."
                $publishResult = m365 spo page publish --webUrl $SiteUrl --pageName $destinationPageName 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Updated but failed to publish '$destinationPageName'. Error: $publishResult"
                }

                Write-Verbose "  Completed '$destinationPageName'."
                $script:Summary.Saved++
            }
        }
        catch {
            Write-Warning "Skipped '$destinationPageName': Unexpected error. $_"
            $script:Summary.Skipped++
            continue
        }
    }
}

end {
    Write-Host "\n========================================" -ForegroundColor Cyan
    Write-Host "Web Part Settings Copy Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Pages saved:   " -NoNewline -ForegroundColor Green
    Write-Host $script:Summary.Saved -ForegroundColor White
    Write-Host "Pages skipped: " -NoNewline -ForegroundColor Yellow
    Write-Host $script:Summary.Skipped -ForegroundColor White
    Write-Host "========================================\n" -ForegroundColor Cyan

    Stop-Transcript
}

<#
# Usage Examples:

# Example 1: Copy web part settings from page-1 to page-2 and page-3
.\Copy-WebPartSettings.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" `
    -WebPartId "544c1372-42df-47c3-94d6-017428cd2baf" `
    -SourcePageName "page-1.aspx" `
    -DestinationPageNames @("page-2.aspx", "page-3.aspx")

# Example 2: Copy from different section/order positions
.\Copy-WebPartSettings.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" `
    -WebPartId "544c1372-42df-47c3-94d6-017428cd2baf" `
    -SourcePageName "template.aspx" `
    -SourceSection 1 `
    -SourceOrder 2 `
    -DestinationPageNames @("page-10.aspx", "page-11.aspx") `
    -DestinationSection 0 `
    -DestinationOrder 1

# Example 3: Use WhatIf to preview changes
.\Copy-WebPartSettings.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" `
    -WebPartId "544c1372-42df-47c3-94d6-017428cd2baf" `
    -SourcePageName "page-1.aspx" `
    -DestinationPageNames @("page-2.aspx", "page-3.aspx") `
    -WhatIf

# Example 4: Verbose output for troubleshooting
.\Copy-WebPartSettings.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" `
    -WebPartId "544c1372-42df-47c3-94d6-017428cd2baf" `
    -SourcePageName "page-1.aspx" `
    -DestinationPageNames @("page-2.aspx") `
    -Verbose
#>

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

    # The site where the web parts are present
    # TODO: Enter value
    $siteUrl = "";

    # The Id of the webpart - see manifest
    # TODO: Enter value 
    $webpartId = ""; # e.g. 544c1372-42df-47c3-94d6-017428cd2baf

    # The name of the page where the web part properties have been updated
    # TODO: Enter value
    $sourcePageName = ""; #e.g. page-1.aspx

    # The section where the web part is present in the source page
    # TODO: Change value
    $sourceWebpartSectionNumber = 0;

    # The coulmn in the section where the web part is present in the source page
    # TODO: Change value
    $sourceWebpartColumnNumber = 0;

    # The order of the web part in the column in the source page
    # TODO: Change value
    $sourceWebpartOrderNumber = 0;

    # The names of the pages where the web part properties need to be updated
    # TODO: Enter values
    $destinationPageNames = @(
        "", # e.g. page-2.aspx
        ""  # e.g. page-3.aspx
    );

    # The section where the web part is present in the destination pages
    # TODO: Change value
    $destinationWebpartSectionNumber = 0;

    # The coulmn in the section where the web part is present in the destination pages
    # TODO: Change value
    $destinationWebpartColumnNumber = 0;

    # The order of the web part in the column in the destination pages
    # TODO: Change value
    $destinationWebpartOrderNumber = 0;

    # Arrays to store results
    $savedPages = @();
    $skippedPages = @();

    # Functions
    function Get-WebpartSettings {
        param(
            [string]$pageName,
            [int]$sectionNumber,
            [int]$columnNumber,
            [int]$controlNumber
        )
        $page = Get-PnPPage $pageName;

        if ($null -eq $page) {
            Write-Error "    Page doesn't exist. Please check the page name.";
            return $null;
        }

        # Change the below based on the type of the webpart
        # as not all web parts will have settings in `PropertiesJson`
        # * Add null checks for section and column if needed
        return $page.Sections[$sectionNumber].Columns[$columnNumber].Controls[$controlNumber].PropertiesJson;
    };

    function Update-WebpartSettings {
        param(
            [string]$pageName,
            [int]$sectionNumber,
            [int]$columnNumber,
            [int]$controlNumber,
            [string]$webPartId,
            $webPartProps
        )

        $page = Get-PnPPage $pageName;

        if ($null -eq $page) {
            Write-Error "    Skipped $pageName as page is null.";
            return $false;
        }

        $webPart = $page.Sections[$sectionNumber].Columns[$columnNumber].Controls[$controlNumber];

        if ($null -eq $webPart) {
            Write-Error "    Skipped $pageName as webpart is null";
            return $false;
        }

        if ($webPart.WebPartId -ne $webPartId) {
            Write-Host "    Skipped $pageName as webpart is different compared to the one specified" -ForegroundColor Yellow;
            return $false;
        }

        # Change the below based on the type of the webpart
        # as not all web parts will have settings in `PropertiesJson`
        # * Add null checks for section and column if needed
        $page.Sections[$sectionNumber].Columns[$columnNumber].Controls[$controlNumber].PropertiesJson = $webPartProps;
            
        $page.Save() | Out-Null;
        $page.Publish();

        Write-Host "    Completed $pageName" -ForegroundColor Green;
        return $true;
    }

    # End functions

    # Start

    if ($null -ne $env:PNPPSSITE) {
        Disconnect-PnPOnline;
    }

    Connect-PnPOnline $siteUrl -UseWebLogin;

    # If there is an error in the connection then exit
    if ($null -eq $env:PNPPSSITE) {
        Write-Error "Not proceeding as there was an error in connecting to the site.";
        exit;
    }

    $sourceWebPartProps = Get-WebpartSettings `
        -pageName $sourcePageName `
        -sectionNumber $sourceWebpartSectionNumber `
        -columnNumber $sourceWebpartColumnNumber `
        -controlNumber $sourceWebpartOrderNumber `;

    if ($null -eq $sourceWebPartProps) {
        Write-Error "Not proceeding as source webpart or it's properties are empty.";
        exit;
    }

    $destinationPageNames | ForEach-Object {
        $destinationPageName = $_;
        Write-Host "    --------------------------      " -ForegroundColor White;
        $destinationWebPartUpdated = Update-WebpartSettings `
            -pageName $destinationPageName `
            -sectionNumber $destinationWebpartSectionNumber `
            -columnNumber $destinationWebpartColumnNumber `
            -controlNumber $destinationWebpartOrderNumber `
            -webPartId $webpartId `
            -webPartProps $sourceWebPartProps;
        Write-Host "    --------------------------      " -ForegroundColor White;

        if ($destinationWebPartUpdated) {
            $savedPages += $destinationPageName;
        }
        else {
            $skippedPages += $destinationPageName;
        }
    }

    Write-Host "    --------------------------      " -ForegroundColor White;
        

    Write-Host "    Saved pages:" -ForegroundColor Green;
    $savedPages | ForEach-Object {
        Write-Host "    $_" -ForegroundColor Green;
    };

    Write-Host "    Skipped pages:" -ForegroundColor Yellow;
    $skippedPages | ForEach-Object {
        Write-Host "    $_" -ForegroundColor Yellow;
    };

    Write-Host "    --------------------------      " -ForegroundColor White;

    Disconnect-PnPOnline;

    # End

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Anoop Tatti |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-copy-webpart-settings" aria-hidden="true" />

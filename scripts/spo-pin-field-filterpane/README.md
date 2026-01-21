

# Pinning Fields to the Filter Pane in SharePoint Libraries

## Summary

The filter pane allows users to quickly filter and find relevant data in libraries and lists. However, by default, not all fields are visible in the filter pane. To enhance usability, specific fields can be pinned to the top of the filter pane.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have access to the SharePoint Online site.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "List title where fields are located")]
    [string]$ListTitle,

    [Parameter(Mandatory = $true, HelpMessage = "Internal names or titles of fields to pin")]
    [string[]]$FieldNames
)

begin {
    Write-Verbose "Authenticating to Microsoft 365..."
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to Microsoft 365"
    }

    $script:Summary = @{
        Total    = $FieldNames.Count
        Pinned   = 0
        Failures = 0
    }

    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    Start-Transcript -Path "PinFields_$timestamp.log"
}

process {
    Write-Host "Pinning $($script:Summary.Total) field(s) to filter pane in list '$ListTitle'..." -ForegroundColor Cyan

    foreach ($fieldName in $FieldNames) {
        $script:Summary.Total = $FieldNames.Count

        try {
            if ($PSCmdlet.ShouldProcess($fieldName, 'Pin field to filter pane')) {
                Write-Verbose "Processing field: $fieldName"

                m365 spo field set --webUrl $SiteUrl --listTitle $ListTitle --title $fieldName --ShowInFiltersPane 1 2>&1 | Out-Null

                if ($LASTEXITCODE -ne 0) {
                    throw "CLI command failed with exit code $LASTEXITCODE"
                }

                Write-Verbose "Successfully pinned field: $fieldName"
                $script:Summary.Pinned++
            } else {
                $script:Summary.Pinned++
            }
        }
        catch {
            Write-Warning "Failed to pin field '$fieldName': $($_.Exception.Message)"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Stop-Transcript

    Write-Host "`n===== Summary =====" -ForegroundColor Cyan
    Write-Host "List: $ListTitle" -ForegroundColor White
    Write-Host "Site: $SiteUrl" -ForegroundColor White
    Write-Host "Total fields attempted: $($script:Summary.Total)" -ForegroundColor White
    Write-Host "Successfully pinned: $($script:Summary.Pinned)" -ForegroundColor Green

    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor White
    }
}

# Example 1: Pin two fields to filter pane
# .\Pin-FieldsToFilterPane.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -ListTitle "Documents" -FieldNames @("Modified", "Editor")

# Example 2: Test with WhatIf (no changes applied)
# .\Pin-FieldsToFilterPane.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -ListTitle "Policies" -FieldNames @("ContentType", "Modified") -WhatIf

# Example 3: Pin multiple fields with verbose output
# .\Pin-FieldsToFilterPane.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Tasks" -FieldNames @("Status", "Priority", "DueDate") -Verbose

# Example 4: Pin single field
# .\Pin-FieldsToFilterPane.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/sales" -ListTitle "Leads" -FieldNames @("Stage")

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell
# This script updates a field within a SharePoint Online library to pin it to the top of the filters pane.

# Parameters
function Pin-FieldsInList() {
    param (
        [string]$SiteUrl,
        [string]$ListTitle,
        [string[]]$FieldNames
    )

    # Connect to SharePoint Online
    Connect-PnPOnline -Url $SiteUrl

    # Loop through each field to update its properties
    foreach ($FieldName in $FieldNames) {
        Write-Host "Updating field '$FieldName' in library '$ListTitle'..."

        # Get the field
        $Field = Get-PnPField -List $ListTitle -Identity $FieldName

        if ($Field) {
            # Update the field to show in the filters pane
            Set-PnPField -List $ListTitle -Identity $FieldName -Values @{ShowInFiltersPane = 1}
            Write-Host "Field '$FieldName' has been pinned to the filters pane."
        } else {
            Write-Host "Field '$FieldName' not found in library '$ListTitle'."
        }
    }
}

# Example how to call the function Pin-FieldsInList 
Pin-FieldsInList -SiteUrl "https://contoso.sharepoint.com/teams/TestMultipleLibraries" -ListTitle "amberlib" -FieldNames @("Modified", "Type")
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Pinning Fields to the Filter Pane in SharePoint Libraries Using PowerShell](https://reshmeeauckloo.com/posts/powershell-sharepoint-library-pin-field-filter-pane/)

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Reshmee Auckloo](https://github.com/reshmee011) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-pin-field-filterpane" aria-hidden="true" />

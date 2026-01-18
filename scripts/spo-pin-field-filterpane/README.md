

# Pinning Fields to the Filter Pane in SharePoint Libraries

## Summary

The filter pane allows users to quickly filter and find relevant data in libraries and lists. However, by default, not all fields are visible in the filter pane. To enhance usability, specific fields can be pinned to the top of the filter pane.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have access to the SharePoint Online site.

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

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$SiteUrl,

    [Parameter(Mandatory, HelpMessage = "Title of the list or library")]
    [string]$ListTitle,

    [Parameter(Mandatory, HelpMessage = "Array of field names to pin to the filter pane")]
    [string[]]$FieldNames,

    [Parameter(HelpMessage = "Path where the CSV report will be saved")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Host "Ensuring CLI for Microsoft 365 authentication..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }

    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath -PathType Container)) {
            throw "Output path does not exist: $OutputPath"
        }
    }

    $script:Summary = @{
        TotalFields    = 0
        FieldsPinned   = 0
        AlreadyPinned  = 0
        Failures       = 0
    }

    $script:ReportCollection = @()

    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "PinFields_Transcript_$timestamp.log"
    Start-Transcript -Path $transcriptPath

    Write-Host "Processing fields in list '$ListTitle' at site '$SiteUrl'..." -ForegroundColor Cyan
}

process {
    foreach ($fieldName in $FieldNames) {
        $script:Summary.TotalFields++
        Write-Host "Processing field: $fieldName" -ForegroundColor Yellow

        try {
            $fieldJson = m365 spo field get --webUrl $SiteUrl --listTitle $ListTitle --title $fieldName --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Field not found or access denied"
            }

            $field = $fieldJson | ConvertFrom-Json
            $currentState = $field.ShowInFiltersPane

            if ($currentState -eq 1) {
                Write-Verbose "Field '$fieldName' is already pinned to the filter pane. Skipping."
                $script:Summary.AlreadyPinned++

                $script:ReportCollection += [PSCustomObject]@{
                    SiteUrl       = $SiteUrl
                    ListTitle     = $ListTitle
                    FieldName     = $fieldName
                    PreviousState = "Already Pinned"
                    NewState      = "Pinned"
                    Status        = "Skipped"
                    ErrorMessage  = ""
                }
                continue
            }

            if ($PSCmdlet.ShouldProcess($fieldName, "Pin field to filter pane")) {
                m365 spo field set --webUrl $SiteUrl --listTitle $ListTitle --title $fieldName --ShowInFiltersPane 1 2>&1 | Out-Null
                if ($LASTEXITCODE -ne 0) {
                    throw "Failed to update field"
                }

                Write-Host "  SUCCESS: Field '$fieldName' has been pinned to the filter pane" -ForegroundColor Green
                $script:Summary.FieldsPinned++

                $script:ReportCollection += [PSCustomObject]@{
                    SiteUrl       = $SiteUrl
                    ListTitle     = $ListTitle
                    FieldName     = $fieldName
                    PreviousState = "Not Pinned"
                    NewState      = "Pinned"
                    Status        = "Success"
                    ErrorMessage  = ""
                }
            } else {
                Write-Host "  WHATIF: Would pin field '$fieldName' to the filter pane" -ForegroundColor Yellow
                $script:Summary.FieldsPinned++

                $script:ReportCollection += [PSCustomObject]@{
                    SiteUrl       = $SiteUrl
                    ListTitle     = $ListTitle
                    FieldName     = $fieldName
                    PreviousState = "Not Pinned"
                    NewState      = "Would Pin (WhatIf)"
                    Status        = "WhatIf"
                    ErrorMessage  = ""
                }
            }
        }
        catch {
            Write-Warning "Failed to process field '$fieldName': $_"
            $script:Summary.Failures++

            $script:ReportCollection += [PSCustomObject]@{
                SiteUrl       = $SiteUrl
                ListTitle     = $ListTitle
                FieldName     = $fieldName
                PreviousState = "Unknown"
                NewState      = "Failed"
                Status        = "Failed"
                ErrorMessage  = $_.ToString()
            }
        }
    }
}

end {
    $csvTimestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $csvPath = Join-Path $OutputPath "PinFields_Report_$csvTimestamp.csv"
    $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
    Write-Host "CSV report exported to: $csvPath" -ForegroundColor Cyan

    Stop-Transcript

    Write-Host "`n==================== SUMMARY ====================" -ForegroundColor Cyan
    Write-Host "Total Fields Processed : $($script:Summary.TotalFields)" -ForegroundColor White
    Write-Host "Fields Pinned          : $($script:Summary.FieldsPinned)" -ForegroundColor Green
    Write-Host "Already Pinned (Skipped): $($script:Summary.AlreadyPinned)" -ForegroundColor Yellow
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures               : $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures               : $($script:Summary.Failures)" -ForegroundColor Green
    }
    Write-Host "================================================" -ForegroundColor Cyan
}

# Usage examples (run one at a time):

# Example 1: Test with WhatIf (safe dry-run before making changes)
# .\Pin-FieldsToFilterPane.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Documents" -FieldNames @("Modified", "Editor") -WhatIf

# Example 2: Pin a single field with Verbose output
# .\Pin-FieldsToFilterPane.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Documents" -FieldNames @("Modified") -Verbose

# Example 3: Pin multiple fields (basic usage)
# .\Pin-FieldsToFilterPane.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Documents" -FieldNames @("Modified", "Editor", "FileType")

# Example 4: Pin fields with custom output path
# .\Pin-FieldsToFilterPane.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "TestLibrary" -FieldNames @("Created", "Author") -OutputPath "C:\Reports"

# Example 5: Pin field to a list (not just library)
# .\Pin-FieldsToFilterPane.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -ListTitle "Employee List" -FieldNames @("Department", "Status")
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

Sample first appeared on [Pinning Fields to the Filter Pane in SharePoint Libraries Using PowerShell](https://reshmeeauckloo.com/posts/powershell-sharepoint-library-pin-field-filter-pane/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-pin-field-filterpane" aria-hidden="true" />

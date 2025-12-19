

# Prevent Guests from being added to a specific Microsoft 365 Group or Microsoft Teams team

## Summary

By default, guest access for Microsoft 365 groups is enabled within the tenant. This can be controlled either to allow or block guest access at the tenant level or for individual Microsoft 365 groups / Microsoft Teams team. This sample is available in both PnP PowerShell and CLI for Microsoft 365. For more information, check out [Manage guest access in Microsoft 365 groups](https://learn.microsoft.com/en-us/microsoft-365/admin/create-groups/manage-guest-access-in-groups?view=o365-worldwide&wt.mc_id=MVP_308367).

This script will enable or disable adding guests to a Microsoft 365 Group or Microsoft Teams team.

![Example Screenshot](assets/example.png)

# [PnP PowerShell](#tab/pnpps)

```powershell
param (
    [Parameter(Mandatory = $true)]
    [string] $domain,
    [Parameter(Mandatory = $true)]
    [ValidateSet("true", "false")]
    [string] $allowToAddGuests
)

$adminSiteURL = "https://$domain-Admin.SharePoint.com"
$dateTime = "_{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "m365_disable_addguests" + $dateTime + ".csv"
$outputPath = $directorypath + "\"+ $fileName

if (-not (Test-Path $outputPath)) {
    New-Item -ItemType File -Path $outputPath
}
Connect-PnPOnline -Url $adminSiteURL -Interactive -WarningAction SilentlyContinue
# amend as required to be the correct filter
$report =  Get-PnPMicrosoft365Group -Filter "startswith(displayName, 'test')" | ForEach-Object {
    $group = $_

    $groupSettings = Get-PnPMicrosoft365GroupSettings -Identity  $group.Id
    if (-Not $groupSettings)
    {
        $groupSettings = New-PnPMicrosoft365GroupSettings -Identity  $group.Id -DisplayName "Group.Unified.Guest" -TemplateId "08d542b9-071f-4e16-94b0-74abb372e3d9" -Values @{"AllowToAddGuests"=$allowToAddGuests}
    }
    if (($groupSettings.Values | Where-Object { $_.Name -eq "AllowToAddGuests"}).Value.ToString() -ne $allowToAddGuests)
    {
        $groupSettings = Set-PnPMicrosoft365GroupSettings -Identity $groupSettings.ID -Group  $group.Id -Values @{"AllowToAddGuests"=$allowToAddGuests}
    }

    #retrieving the details to ensure the settings are applied
    $groupSettings =  Get-PnPMicrosoft365GroupSettings -Identity  $group.Id
    $allowToAddGuestsValue = ($groupSettings.Values | Where-Object { $_.Name -eq "AllowToAddGuests"}).Value.ToString()
     [PSCustomObject]@{
        id = $group.Id
        Description = $group.Description
        DisplayName = $group.DisplayName
        m365GroupAllowToAddGuests = $allowToAddGuestsValue ?? "Default"
    }
}
$report |select *  |Export-Csv $outputPath -NoTypeInformation -Append
Disconnect-PnPOnline
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
<#
.SYNOPSIS
    Controls guest access (AllowToAddGuests) for Microsoft 365 Groups filtered by display name.

.DESCRIPTION
    This script enables or disables the ability to add guests to Microsoft 365 Groups that match a specified filter.
    Uses CLI for Microsoft 365 to retrieve groups and Microsoft Graph API (via m365 request) to manage group-specific settings.
    Supports WhatIf and Verbose modes. Results can be exported to CSV or displayed in the terminal.
    Failed operations are tracked in the summary and included in the report for troubleshooting.

.PARAMETER DisplayNameFilter
    JMESPath filter for group display names (e.g., "startswith(displayName, 'test')" or "contains(displayName, 'HR')").

.PARAMETER AllowToAddGuests
    Whether to allow adding guests to the groups. Valid values: 'true' or 'false'.

.PARAMETER OutputPath
    Path where the CSV file will be saved. Defaults to current directory with timestamp.

.PARAMETER ExportToCsv
    If specified, exports results to a CSV file. Otherwise, displays results in the terminal.

.EXAMPLE
    .\Control-GuestAccess.ps1 -DisplayNameFilter "startswith(displayName, 'test')" -AllowToAddGuests false -Verbose
    Disables guest access for groups starting with 'test' and displays results in terminal with verbose output.

.EXAMPLE
    .\Control-GuestAccess.ps1 -DisplayNameFilter "startswith(displayName, 'HR')" -AllowToAddGuests true -ExportToCsv -Verbose
    Enables guest access for groups starting with 'HR' and exports results to timestamped CSV file.

.EXAMPLE
    .\Control-GuestAccess.ps1 -DisplayNameFilter "contains(displayName, 'Finance')" -AllowToAddGuests false -OutputPath "C:\\Reports\\finance.csv" -ExportToCsv
    Disables guest access for groups containing 'Finance' and exports to custom path.

.EXAMPLE
    .\Control-GuestAccess.ps1 -DisplayNameFilter "displayName eq 'Marketing Team'" -AllowToAddGuests true -ExportToCsv -Verbose
    Enables guest access for a single group by exact name match.

.EXAMPLE
    .\Control-GuestAccess.ps1 -DisplayNameFilter "startswith(displayName, 'Dev')" -AllowToAddGuests false -WhatIf -Verbose
    Tests what changes would be made without applying them (WhatIf mode).
#>

[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "JMESPath filter for group display names (e.g., 'startswith(displayName, \"test\")'")]
    [string]$DisplayNameFilter,

    [Parameter(Mandatory = $true, HelpMessage = "Whether to allow adding guests ('true' or 'false')")]
    [ValidateSet('true', 'false')]
    [string]$AllowToAddGuests,

    [Parameter(HelpMessage = "Path where the CSV file will be saved")]
    [string]$OutputPath = "m365_group_guest_settings_$(Get-Date -Format 'MM-dd-yyyy-HH-mm-ss').csv",

    [Parameter(HelpMessage = "Export results to CSV file")]
    [switch]$ExportToCsv
)

begin {
    Write-Verbose "Starting guest access control script"

    if ($ExportToCsv) {
        $directory = Split-Path -Path $OutputPath -Parent
        if ($directory -and -not (Test-Path -Path $directory)) {
            throw "Directory does not exist: $directory"
        }
    }

    Write-Verbose "Verifying CLI for Microsoft 365 connection"
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to verify CLI for Microsoft 365 connection. Please run 'm365 login' first."
    }
    Write-Verbose "CLI for Microsoft 365 connection verified"

    $script:Summary = @{
        GroupsFound = 0
        SettingsUpdated = 0
        SettingsCreated = 0
        NoChangeNeeded = 0
        Failures = 0
    }

    $script:Report = [System.Collections.ArrayList]::new()
    $script:TemplateId = "08d542b9-071f-4e16-94b0-74abb372e3d9"  # Group.Unified.Guest
}

process {
    Write-Verbose "Retrieving Microsoft 365 Groups with filter: $DisplayNameFilter"
    $groupsJson = m365 entra m365group list --query "[$DisplayNameFilter]" --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve groups. CLI output: $groupsJson"
    }

    $groups = @($groupsJson | ConvertFrom-Json)
    $script:Summary.GroupsFound = $groups.Count
    Write-Verbose "Found $($groups.Count) group(s) matching filter"

    if ($groups.Count -eq 0) {
        Write-Warning "No groups found matching filter: $DisplayNameFilter"
        return
    }

    foreach ($group in $groups) {
        Write-Verbose "Processing group: $($group.displayName) (ID: $($group.id))"

        try {
            Write-Verbose "  Retrieving group settings"
            $settingsJson = m365 request --url "https://graph.microsoft.com/v1.0/groups/$($group.id)/settings" --method GET --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve settings for group '$($group.displayName)'. CLI output: $settingsJson"
                $script:Summary.Failures++
                [void]$script:Report.Add([PSCustomObject]@{
                    GroupId = $group.id
                    DisplayName = $group.displayName ?? "(No Display Name)"
                    Description = $group.description
                    PreviousValue = "Error"
                    AllowToAddGuests = "Error"
                    Status = "Failed to retrieve settings"
                    ErrorMessage = "$settingsJson"
                })
                continue
            }

            $settingsResponse = $settingsJson | ConvertFrom-Json
            $existingSettings = $settingsResponse.value | Where-Object { $_.templateId -eq $script:TemplateId }

            $settingApplied = $false
            $currentValue = "Default"

            if (-not $existingSettings) {
                Write-Verbose "  No settings found, creating new settings with AllowToAddGuests=$AllowToAddGuests"
                if ($PSCmdlet.ShouldProcess($group.displayName, "Create group settings with AllowToAddGuests=$AllowToAddGuests")) {
                    $body = @{
                        templateId = $script:TemplateId
                        values = @(
                            @{ name = "AllowToAddGuests"; value = $AllowToAddGuests }
                        )
                    } | ConvertTo-Json -Depth 10

                    $createJson = m365 request --url "https://graph.microsoft.com/v1.0/groups/$($group.id)/settings" --method POST --body $body --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to create settings for group '$($group.displayName)'. CLI output: $createJson"
                        $script:Summary.Failures++
                        [void]$script:Report.Add([PSCustomObject]@{
                            GroupId = $group.id
                            DisplayName = $group.displayName ?? "(No Display Name)"
                            Description = $group.description
                            PreviousValue = $currentValue
                            AllowToAddGuests = "Error"
                            Status = "Failed to create settings"
                            ErrorMessage = "$createJson"
                        })
                        continue
                    }
                    $script:Summary.SettingsCreated++
                    $currentValue = "Default"
                    $settingApplied = $true
                    Write-Verbose "  Settings created successfully"
                } else {
                    Write-Verbose "  WhatIf: Would create settings"
                }
            } else {
                # Check if update is needed
                $currentSetting = $existingSettings.values | Where-Object { $_.name -eq "AllowToAddGuests" }
                $currentValue = $currentSetting.value

                if ($currentValue -ne $AllowToAddGuests) {
                    Write-Verbose "  Current value: $currentValue, updating to: $AllowToAddGuests"
                    if ($PSCmdlet.ShouldProcess($group.displayName, "Update AllowToAddGuests from $currentValue to $AllowToAddGuests")) {
                        $body = @{
                            values = @(
                                @{ name = "AllowToAddGuests"; value = $AllowToAddGuests }
                            )
                        } | ConvertTo-Json -Depth 10

                        $updateJson = m365 request --url "https://graph.microsoft.com/v1.0/groups/$($group.id)/settings/$($existingSettings.id)" --method PATCH --body $body --output json 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to update settings for group '$($group.displayName)'. CLI output: $updateJson"
                            $script:Summary.Failures++
                            continue
                        }
                        $script:Summary.SettingsUpdated++
                        $settingApplied = $true
                        Write-Verbose "  Settings updated successfully"
                    } else {
                        Write-Verbose "  WhatIf: Would update settings"
                    }
                } else {
                    Write-Verbose "  Current value matches desired value, no update needed"
                    $script:Summary.NoChangeNeeded++
                }
            }

            $verifyJson = m365 request --url "https://graph.microsoft.com/v1.0/groups/$($group.id)/settings" --method GET --output json 2>&1
            if ($LASTEXITCODE -eq 0) {
                $verifyResponse = $verifyJson | ConvertFrom-Json
                $finalSettings = $verifyResponse.value | Where-Object { $_.templateId -eq $script:TemplateId }
                if ($finalSettings) {
                    $finalValue = ($finalSettings.values | Where-Object { $_.name -eq "AllowToAddGuests" }).value
                } else {
                    $finalValue = "Default"
                }
            } else {
                $finalValue = "Unknown"
            }

            [void]$script:Report.Add([PSCustomObject]@{
                GroupId = $group.id
                DisplayName = $group.displayName ?? "(No Display Name)"
                Description = $group.description
                PreviousValue = $currentValue
                AllowToAddGuests = $finalValue
                Status = if ($settingApplied) { "Updated" } elseif ($currentValue -eq $AllowToAddGuests) { "No change needed" } else { "WhatIf mode" }
                ErrorMessage = ""
            })

        } catch {
            Write-Warning "Error processing group '$($group.displayName)': $($_.Exception.Message)"
            $script:Summary.Failures++
            [void]$script:Report.Add([PSCustomObject]@{
                GroupId = $group.id
                DisplayName = $group.displayName ?? "(No Display Name)"
                Description = $group.description
                PreviousValue = "Error"
                AllowToAddGuests = "Error"
                Status = "Exception"
                ErrorMessage = $_.Exception.Message
            })
        }
    }
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Guest Access Control Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Groups found:         $($script:Summary.GroupsFound)" -ForegroundColor White
    Write-Host "Settings created:     $($script:Summary.SettingsCreated)" -ForegroundColor Green
    Write-Host "Settings updated:     $($script:Summary.SettingsUpdated)" -ForegroundColor Green
    Write-Host "No change needed:     $($script:Summary.NoChangeNeeded)" -ForegroundColor Yellow
    Write-Host "Failures:             $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "========================================`n" -ForegroundColor Cyan

    if ($ExportToCsv -and $script:Report.Count -gt 0) {
        if ($PSCmdlet.ShouldProcess($OutputPath, "Export results to CSV")) {
            $script:Report | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
            Write-Host "Results exported to: $OutputPath" -ForegroundColor Green
        }
    } elseif ($script:Report.Count -gt 0) {
        Write-Host "Group Guest Access Settings:" -ForegroundColor Yellow
        $script:Report | Format-Table -AutoSize
    }

    Write-Verbose "Script execution completed"
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]


***

## Source Credit

Sample first appeared on [Prevent Guests from Being Added to a Specific Microsoft 365 Group or Microsoft Teams team using PnP PowerShell](https://reshmeeauckloo.com/posts/powershell-m365Group-disable-add-guests/)


## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it)|


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/aad-control-guestaccount-m365-groups-teams" aria-hidden="true" />

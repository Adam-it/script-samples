

# Update Global Microsoft 365 Group Settings

## Summary

Managing Microsoft 365 Group settings is crucial for maintaining a compliant and secure environment. PowerShell and Microsoft Graph can be used to configure various group settings, including naming policies, guest access, and more. This sample is available in both PnP PowerShell and CLI for Microsoft 365.

As a regular user of PnP PowerShell, I wanted to replicate the [Microsoft Entra cmdlets for configuring group settings](https://learn.microsoft.com/en-us/entra/identity/users/groups-settings-cmdlets?wt.mc_id=MVP_308367) using PnP PowerShell.

### Prerequisites

- PnP PowerShell https://pnp.github.io/powershell/
- The user account that runs the script must have Global Admin administrator access or Entra ID Admin role.

# [PnP PowerShell](#tab/pnpps)

```powershell
param (
    [Parameter(Mandatory = $true)]
    [string] $domain
)
Clear-Host

$adminSiteURL = "https://$domain-Admin.SharePoint.com"
Connect-PnPOnline -Url $adminSiteURL

$BlockedWords = @("law","pension","finance","sea","ocean","river","lake","stream","creek","pond","pool","reservoir","dam","canal","ditch","drain","gutter","sewer","pipe","tube","hose","conduit","channel","aqua")

$unifiedSet = Get-PnPMicrosoft365GroupSettings -GroupSetting "Group.Unified"

$url = "/groupSettings/$($unifiedSet.Id)"
  
$Payload = @"
{
    "values": [
        {
            "name": "CustomBlockedWordsList",
            "value": "$($BlockedWords -join ',')"
        },
        {
            "name": "PrefixSuffixNamingRequirement",
            "value": "Test_[Department][GroupName][Office]"
        }
        ,
        {
            "name": "AllowToAddGuests",
            "value": "True"
        },
        {
            "name": "AllowGuestsToBeGroupOwner",
            "value": "False"
        },
        {
            "name": "AllowGuestsToAccessGroups",
            "value": "True"
        },
        {
            "name": "AllowToAddGuests",
            "value": "True"
        },
        {
            "name": "EnableGroupCreation",
            "value": "True"
        },
        {
            "name": "NewUnifiedGroupWritebackDefault",
            "value": "True"
        },
        {
            "name": "EnableMIPLabels",
            "value": "True"
        },
        {
            "name": "EnableMSStandardBlockedWords",
            "value": "False"
        }
     ]
}
"@


Invoke-PnPGraphMethod -Url $url -Method Patch -Content $Payload
(Get-PnPMicrosoft365GroupSettings -GroupSetting "Group.Unified").Values 
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(HelpMessage = "Comma-separated list of blocked words")]
    [string]$CustomBlockedWordsList,
    
    [Parameter(HelpMessage = "Naming pattern (e.g., 'Test_[Department][GroupName][Office]')")]
    [string]$PrefixSuffixNamingRequirement,
    
    [Parameter(HelpMessage = "Allow members to add guests (true/false)")]
    [ValidateSet("true", "false")]
    [string]$AllowToAddGuests,
    
    [Parameter(HelpMessage = "Allow guests to be group owners (true/false)")]
    [ValidateSet("true", "false")]
    [string]$AllowGuestsToBeGroupOwner,
    
    [Parameter(HelpMessage = "Allow guests to access groups (true/false)")]
    [ValidateSet("true", "false")]
    [string]$AllowGuestsToAccessGroups,
    
    [Parameter(HelpMessage = "Enable group creation (true/false)")]
    [ValidateSet("true", "false")]
    [string]$EnableGroupCreation,
    
    [Parameter(HelpMessage = "Enable MIP labels (true/false)")]
    [ValidateSet("true", "false")]
    [string]$EnableMIPLabels,
    
    [Parameter(HelpMessage = "Enable Microsoft standard blocked words (true/false)")]
    [ValidateSet("true", "false")]
    [string]$EnableMSStandardBlockedWords,
    
    [Parameter(HelpMessage = "Enable group writeback to AD (true/false)")]
    [ValidateSet("true", "false")]
    [string]$NewUnifiedGroupWritebackDefault
)

begin {
    Write-Verbose "Starting M365 Group settings update script"
    
    Write-Verbose "Verifying CLI for Microsoft 365 connection"
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to verify CLI for Microsoft 365 connection. Please run 'm365 login' first."
    }
    Write-Verbose "CLI for Microsoft 365 connection verified"
    
    Write-Verbose "Retrieving Group.Unified settings"
    $settingsJson = m365 entra groupsetting list --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve group settings. CLI output: $settingsJson"
    }
    
    $allSettings = @($settingsJson | ConvertFrom-Json)
    $script:unifiedSetting = $allSettings | Where-Object { $_.displayName -eq "Group.Unified" }
    
    if ($script:unifiedSetting) {
        Write-Verbose "Found existing Group.Unified setting with ID: $($script:unifiedSetting.id)"
        Write-Host "`n========================================" -ForegroundColor Cyan
        Write-Host "Current Group.Unified Settings" -ForegroundColor Cyan
        Write-Host "========================================" -ForegroundColor Cyan
        $script:unifiedSetting.values | ForEach-Object {
            Write-Host "$($_.name): $($_.value)" -ForegroundColor White
        }
        Write-Host "========================================`n" -ForegroundColor Cyan
    } else {
        Write-Verbose "Group.Unified setting does not exist. Will create it in process block."
    }
}

process {
    if ($null -eq $script:unifiedSetting) {
        Write-Warning "Group.Unified setting does not exist. Creating it now..."
        
        $createParams = @('entra', 'groupsetting', 'add', '--templateId', '62375ab9-6b52-47ed-826b-58e47e0e304b')
        
        if ($PSBoundParameters.ContainsKey('CustomBlockedWordsList')) { $createParams += @('--CustomBlockedWordsList', $CustomBlockedWordsList) }
        if ($PSBoundParameters.ContainsKey('PrefixSuffixNamingRequirement')) { $createParams += @('--PrefixSuffixNamingRequirement', $PrefixSuffixNamingRequirement) }
        if ($PSBoundParameters.ContainsKey('AllowToAddGuests')) { $createParams += @('--AllowToAddGuests', $AllowToAddGuests) }
        if ($PSBoundParameters.ContainsKey('AllowGuestsToBeGroupOwner')) { $createParams += @('--AllowGuestsToBeGroupOwner', $AllowGuestsToBeGroupOwner) }
        if ($PSBoundParameters.ContainsKey('AllowGuestsToAccessGroups')) { $createParams += @('--AllowGuestsToAccessGroups', $AllowGuestsToAccessGroups) }
        if ($PSBoundParameters.ContainsKey('EnableGroupCreation')) { $createParams += @('--EnableGroupCreation', $EnableGroupCreation) }
        if ($PSBoundParameters.ContainsKey('EnableMIPLabels')) { $createParams += @('--EnableMIPLabels', $EnableMIPLabels) }
        if ($PSBoundParameters.ContainsKey('EnableMSStandardBlockedWords')) { $createParams += @('--EnableMSStandardBlockedWords', $EnableMSStandardBlockedWords) }
        if ($PSBoundParameters.ContainsKey('NewUnifiedGroupWritebackDefault')) { $createParams += @('--NewUnifiedGroupWritebackDefault', $NewUnifiedGroupWritebackDefault) }
        
        $createParams += @('--output', 'json')
        
        if ($PSCmdlet.ShouldProcess("Group.Unified", "Create group setting")) {
            Write-Verbose "Creating Group.Unified setting with specified parameters"
            $createResult = m365 @createParams 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to create Group.Unified setting. CLI output: $createResult"
            }
            $script:unifiedSetting = $createResult | ConvertFrom-Json
            Write-Verbose "Group.Unified setting created successfully with ID: $($script:unifiedSetting.id)"
        }
    } else {
        $hasChanges = $false
        $updateParams = @('entra', 'groupsetting', 'set', '--id', $script:unifiedSetting.id)
        
        if ($PSBoundParameters.ContainsKey('CustomBlockedWordsList')) {
            $updateParams += @('--CustomBlockedWordsList', $CustomBlockedWordsList)
            $hasChanges = $true
            Write-Verbose "Updating CustomBlockedWordsList to: $CustomBlockedWordsList"
        }
        if ($PSBoundParameters.ContainsKey('PrefixSuffixNamingRequirement')) {
            $updateParams += @('--PrefixSuffixNamingRequirement', $PrefixSuffixNamingRequirement)
            $hasChanges = $true
            Write-Verbose "Updating PrefixSuffixNamingRequirement to: $PrefixSuffixNamingRequirement"
        }
        if ($PSBoundParameters.ContainsKey('AllowToAddGuests')) {
            $updateParams += @('--AllowToAddGuests', $AllowToAddGuests)
            $hasChanges = $true
            Write-Verbose "Updating AllowToAddGuests to: $AllowToAddGuests"
        }
        if ($PSBoundParameters.ContainsKey('AllowGuestsToBeGroupOwner')) {
            $updateParams += @('--AllowGuestsToBeGroupOwner', $AllowGuestsToBeGroupOwner)
            $hasChanges = $true
            Write-Verbose "Updating AllowGuestsToBeGroupOwner to: $AllowGuestsToBeGroupOwner"
        }
        if ($PSBoundParameters.ContainsKey('AllowGuestsToAccessGroups')) {
            $updateParams += @('--AllowGuestsToAccessGroups', $AllowGuestsToAccessGroups)
            $hasChanges = $true
            Write-Verbose "Updating AllowGuestsToAccessGroups to: $AllowGuestsToAccessGroups"
        }
        if ($PSBoundParameters.ContainsKey('EnableGroupCreation')) {
            $updateParams += @('--EnableGroupCreation', $EnableGroupCreation)
            $hasChanges = $true
            Write-Verbose "Updating EnableGroupCreation to: $EnableGroupCreation"
        }
        if ($PSBoundParameters.ContainsKey('EnableMIPLabels')) {
            $updateParams += @('--EnableMIPLabels', $EnableMIPLabels)
            $hasChanges = $true
            Write-Verbose "Updating EnableMIPLabels to: $EnableMIPLabels"
        }
        if ($PSBoundParameters.ContainsKey('EnableMSStandardBlockedWords')) {
            $updateParams += @('--EnableMSStandardBlockedWords', $EnableMSStandardBlockedWords)
            $hasChanges = $true
            Write-Verbose "Updating EnableMSStandardBlockedWords to: $EnableMSStandardBlockedWords"
        }
        if ($PSBoundParameters.ContainsKey('NewUnifiedGroupWritebackDefault')) {
            $updateParams += @('--NewUnifiedGroupWritebackDefault', $NewUnifiedGroupWritebackDefault)
            $hasChanges = $true
            Write-Verbose "Updating NewUnifiedGroupWritebackDefault to: $NewUnifiedGroupWritebackDefault"
        }
        
        if (-not $hasChanges) {
            Write-Warning "No parameters specified for update. Current settings displayed above."
            return
        }
        
        if ($PSCmdlet.ShouldProcess("Group.Unified", "Update group settings")) {
            Write-Verbose "Updating Group.Unified settings"
            m365 @updateParams 2>&1 | Out-Null
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to update Group.Unified settings"
            }
            Write-Verbose "Settings updated successfully"
        }
    }
    
    Write-Verbose "Retrieving updated settings for verification"
    $verifyJson = m365 entra groupsetting get --id $script:unifiedSetting.id --output json 2>&1
    if ($LASTEXITCODE -eq 0) {
        $script:updatedSettings = $verifyJson | ConvertFrom-Json
        Write-Verbose "Settings retrieved successfully"
    } else {
        Write-Warning "Failed to retrieve updated settings for verification: $verifyJson"
    }
}

end {
    if ($script:updatedSettings) {
        Write-Host "`n========================================" -ForegroundColor Green
        Write-Host "Group Settings Update Complete" -ForegroundColor Green
        Write-Host "========================================" -ForegroundColor Green
        
        Write-Host "`nUpdated Group.Unified Settings:" -ForegroundColor Yellow
        $script:updatedSettings.values | ForEach-Object {
            $propertyName = $_.name
            $newValue = $_.value
            
            if ($PSBoundParameters.ContainsKey($propertyName)) {
                Write-Host "  $propertyName: $newValue" -ForegroundColor Green
            } else {
                Write-Host "  $propertyName: $newValue" -ForegroundColor White
            }
        }
        
        $changedProperties = @()
        foreach ($param in $PSBoundParameters.Keys) {
            if ($param -notin @('Verbose', 'WhatIf', 'Confirm')) {
                $changedProperties += $param
            }
        }
        
        if ($changedProperties.Count -gt 0) {
            Write-Host "`nChanged properties: $($changedProperties -join ', ')" -ForegroundColor Cyan
        }
        
        Write-Host "========================================`n" -ForegroundColor Green
    } else {
        Write-Warning "No settings to display. Operation may have failed or been skipped."
    }
    
    Write-Verbose "Script execution completed"
}

<#
USAGE EXAMPLES:

# Example 1: Update blocked words and naming policy
.\Update-M365GroupSettings.ps1 -CustomBlockedWordsList "law,pension,finance" -PrefixSuffixNamingRequirement "Test_[Department][GroupName][Office]" -Verbose

# Example 2: Disable all guest access
.\Update-M365GroupSettings.ps1 -AllowToAddGuests "false" -AllowGuestsToBeGroupOwner "false" -AllowGuestsToAccessGroups "false" -Verbose

# Example 3: Enable MIP labels and group writeback
.\Update-M365GroupSettings.ps1 -EnableMIPLabels "true" -NewUnifiedGroupWritebackDefault "true" -Verbose

# Example 4: View current settings without changes
.\Update-M365GroupSettings.ps1 -Verbose

# Example 5: Disable group creation (dry-run)
.\Update-M365GroupSettings.ps1 -EnableGroupCreation "false" -WhatIf

# Example 6: Update multiple settings at once
.\Update-M365GroupSettings.ps1 -CustomBlockedWordsList "confidential,secret,private" -EnableMSStandardBlockedWords "true" -AllowToAddGuests "false" -EnableGroupCreation "true" -Verbose
#>
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

Sample first appeared on [Managing Microsoft 365 Group Settings with PnP PowerShell and Microsoft Graph](https://reshmeeauckloo.com/posts/powershell-m365-groupsetting-graph/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/aad-update-m365-global-unified-settings" aria-hidden="true" />

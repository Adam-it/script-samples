

# Export all flows in environment

## Summary

When was the last time you backed up all the flows in your environment?

By combining the CLI for Microsoft 365 and PowerShell along with a new pure PnP PowerShell example we can make this task easy and repeatable.

This script will get all flows in your default environment and export them as both a ZIP file for importing back into Power Automate and as a JSON file for importing into Azure as an Azure Logic App. The CLI version below mirrors the PnP example using `m365 flow` commands.


# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
function Export-FlowEnvironmentBackup {
    [CmdletBinding(SupportsShouldProcess)]
    param (
        [Parameter(Mandatory = $false, HelpMessage = "Target environment display name (default environment when omitted).")]
        [string]$EnvironmentDisplayName,

        [Parameter(Mandatory = $false, HelpMessage = "Limit scope to flows in which the current user has access (no admin permissions).")]
        [switch]$UserScope,

        [Parameter(Mandatory = $false, HelpMessage = "Destination folder for exported packages (defaults to current dir).")]
        [string]$OutputFolder = (Get-Location).Path,

        [Parameter(Mandatory = $false, HelpMessage = "When set, skip exporting individual flows as JSON.")]
        [switch]$SkipJsonExport
    )

    begin {
        Write-Verbose 'Ensuring CLI for Microsoft 365 authentication'
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to sign in with CLI for Microsoft 365. CLI output: $loginOutput"
        }

        if (-not (Test-Path -Path $OutputFolder -PathType Container)) {
            Write-Verbose "Creating output directory $OutputFolder"
            New-Item -ItemType Directory -Path $OutputFolder -Force | Out-Null
        }

        $script:Summary = [ordered]@{
            Environments     = @()
            TotalFlows       = 0
            Exported         = 0
            JsonExported     = 0
            Failures         = 0
            Skipped          = 0
        }

        if ($EnvironmentDisplayName) {
            $escaped = $EnvironmentDisplayName.ToLowerInvariant().Replace("'", "\'")
            $script:EnvironmentQuery = "[?contains(to_lower(properties.displayName), '$escaped')]"
        }
        else {
            $script:EnvironmentQuery = "[?properties.isDefault]"
        }
    }

    process {
        Write-Verbose 'Retrieving environments'
        $envOutput = m365 flow environment list --output json --query $script:EnvironmentQuery 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list environments. CLI output: $envOutput"
        }

        $environments = @()
        if (-not [string]::IsNullOrWhiteSpace($envOutput)) {
            $environments = @($envOutput | ConvertFrom-Json)
        }
        if (-not $environments) {
            throw "No environments found. Ensure the display name is correct and you have permissions."
        }

        foreach ($environment in $environments) {
            $environmentName = $environment.name
            $displayName = $environment.properties.displayName
            Write-Verbose "Processing environment '$displayName' ($environmentName)"

            $flowCommand = @(
                'flow', 'list',
                '--environmentName', $environmentName,
                '--output', 'json'
            )
            if (-not $UserScope.IsPresent) {
                $flowCommand += '--asAdmin'
            }

            $flowsOutput = m365 @flowCommand 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to list flows for environment '$displayName'. CLI output: $flowsOutput"
                $script:Summary.Failures++
                continue
            }

            $flows = @()
            if (-not [string]::IsNullOrWhiteSpace($flowsOutput)) {
                $flows = @($flowsOutput | ConvertFrom-Json)
            }
            $script:Summary.TotalFlows += $flows.Count
            $envRecord = [pscustomobject]@{
                EnvironmentName = $environmentName
                DisplayName     = $displayName
                FlowCount       = $flows.Count
            }
            $script:Summary.Environments += $envRecord

            foreach ($flow in $flows) {
                $flowDisplayName = $flow.properties.displayName
                $flowName = $flow.name
                $safeName = ($flowDisplayName -replace "[^A-Za-z0-9_-]", '_')
                $timestamp = Get-Date -Format 'yyyyMMddHHmmss'
                $zipPath = Join-Path -Path $OutputFolder -ChildPath "${safeName}_${timestamp}.zip"
                $jsonPath = Join-Path -Path $OutputFolder -ChildPath "${safeName}_${timestamp}.json"

                if ($PSCmdlet.ShouldProcess($flowDisplayName, 'Export flow as ZIP package')) {
                    $exportZipOutput = m365 flow export `
                        --environmentName $environmentName `
                        --name $flowName `
                        --packageDisplayName "$flowDisplayName" `
                        --format zip `
                        --path $zipPath 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to export flow '$flowDisplayName' as ZIP. CLI output: $exportZipOutput"
                        $script:Summary.Failures++
                        continue
                    }

                    $script:Summary.Exported++
                }
                else {
                    $script:Summary.Skipped++
                    continue
                }

                if (-not $SkipJsonExport.IsPresent) {
                    if ($PSCmdlet.ShouldProcess($flowDisplayName, 'Export flow as JSON definition')) {
                        $exportJsonOutput = m365 flow export `
                            --environmentName $environmentName `
                            --name $flowName `
                            --format json `
                            --path $jsonPath 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to export flow '$flowDisplayName' as JSON. CLI output: $exportJsonOutput"
                        }
                        else {
                            $script:Summary.JsonExported++
                        }
                    }
                }
            }
        }
    }

    end {
        Write-Host '--- Export Summary ---'
        foreach ($env in $script:Summary.Environments) {
            Write-Host ("Environment       : {0} ({1})" -f $env.DisplayName, $env.EnvironmentName)
            Write-Host ("Flows discovered   : {0}" -f $env.FlowCount)
        }

        Write-Host "ZIP exports        : $($script:Summary.Exported)"
        Write-Host "JSON exports       : $($script:Summary.JsonExported)"
        Write-Host "Skipped (WhatIf)   : $($script:Summary.Skipped)"
        Write-Host "Failures           : $($script:Summary.Failures)"

        return [pscustomobject]$script:Summary
    }
}

Export-FlowEnvironmentBackup -Verbose

```

# [PnP PowerShell](#tab/pnpps)
```powershell
$environmentName = "Personal Productivity"

$FlowEnv = Get-PnPFlowEnvironment | Where-Object { $_.Properties.DisplayName -eq $environmentName }

Write-Host "Getting All Flows in $environmentName Environment"
$flows = Get-PnPFlow -Environment $FlowEnv -AsAdmin #Remove -AsAdmin Parameter to only target Flows you have permission to access

Write-Host "Found $($flows.Count) Flows to export..."

foreach ($flow in $flows) {

    Write-Host "Exporting as ZIP & JSON... $($flow.Properties.DisplayName)"
    $filename = $flow.Properties.DisplayName.Replace(" ", "")
    $timestamp = Get-Date -Format "yyyymmddhhmmss"
    $exportPath = "$($filename)_$($timestamp)"
    $exportPath = $exportPath.Split([IO.Path]::GetInvalidFileNameChars()) -join '_'
    Export-PnPFlow -Environment $FlowEnv -Identity $flow.Name -PackageDisplayName $flow.Properties.DisplayName -AsZipPackage -OutPath "$exportPath.zip" -Force
    Export-PnPFlow -Environment $FlowEnv -Identity $flow.Name | Out-File "$exportPath.json"

}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Source Credit

Added PnP PowerShell version from [PnP.PowerShell + Bonus Script: Export all Flows using PnP.PowerShell](https://www.leonarmston.com/2021/01/testing-out-the-new-power-automate-flow-commands-in-pnp-powershell-bonus-script-export-all-flows-using-pnp-powershell/)
## Contributors

| Author(s) |
|-----------|
| Luise Freese |
| [Leon Armston](https://github.com/LeonArmston) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/flow-export-all-flows-in-environment" aria-hidden="true" />

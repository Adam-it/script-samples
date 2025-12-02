

# Flow run status list dashboard

## Summary

Powershell script that reports the status of the latest run of all flows by writing to a M365 list. Shows Title, Status and last run time

Result in console

![run in console](assets/example2.png)

Result in the list

![adaptive card in teams](assets/example.png)
 
# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
function Update-FlowRunDashboard {
    [CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'Medium')]
    param(
        [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL hosting the dashboard list")]
        [ValidateNotNullOrEmpty()]
        [string] $SiteUrl,

        [Parameter(Mandatory = $true, HelpMessage = "Name of the SharePoint list used for the dashboard")]
        [ValidateNotNullOrEmpty()]
        [string] $ListTitle,

        [Parameter(HelpMessage = "Power Platform environment name (defaults to the tenant's Default environment when omitted)")]
        [string] $EnvironmentName,

        [Parameter(HelpMessage = "Include flows that are stopped or disabled")]
        [switch] $IncludeStoppedFlows,

        [Parameter(HelpMessage = "Export dashboard data to a timestamped CSV file in the current directory")]
        [switch] $ExportCsv,

        [switch] $Force
    )

    begin {
        Write-Verbose "Ensuring CLI authentication"
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to sign in to CLI for Microsoft 365. CLI output: $loginOutput"
        }

        $summary = [ordered]@{
            FlowsDiscovered = 0
            FlowsProcessed  = 0
            SkippedStopped  = 0
            SkippedNoRuns   = 0
            ItemsAdded      = 0
            ItemsUpdated    = 0
            Errors          = 0
        }

        $results = [System.Collections.Generic.List[psobject]]::new()

        if (-not $EnvironmentName) {
            Write-Verbose "Resolving flow environment automatically"
            $envJson = m365 flow environment list --output json --query "[].{Name:name, DisplayName:displayName}" 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve environments. CLI output: $envJson"
            }

            try {
                $environments = @($envJson | ConvertFrom-Json)
            }
            catch {
                throw "Failed to parse environment list. Error: $($_.Exception.Message)"
            }

            if ($environments.Count -eq 0) {
                throw "No environments returned by CLI."
            }

            if ($environments.Count -eq 1) {
                $script:resolvedEnvironmentName = $environments[0].Name
            }
            else {
                $defaultEnv = $environments | Where-Object { $_.Name -like 'Default-*' } | Select-Object -First 1
                if ($defaultEnv) {
                    Write-Verbose "Multiple environments detected. Using '$($defaultEnv.DisplayName)' by default."
                    $script:resolvedEnvironmentName = $defaultEnv.Name
                }
                else {
                    $envNames = ($environments | ForEach-Object { $_.Name }) -join ', '
                    throw "Multiple environments found ($envNames). Specify -EnvironmentName to choose one."
                }
            }
        }
        else {
            $script:resolvedEnvironmentName = $EnvironmentName
        }

        Write-Verbose "Retrieving existing dashboard items"
        $existingJson = m365 spo listitem list --title $ListTitle --webUrl $SiteUrl --output json --query '[].{Id:Id,Title:Title}' 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve current dashboard entries. CLI output: $existingJson"
        }

        try {
            $existingItems = if ([string]::IsNullOrWhiteSpace($existingJson)) { @() } else { @($existingJson | ConvertFrom-Json) }
        }
        catch {
            throw "Failed to parse current dashboard entries. Error: $($_.Exception.Message)"
        }

        $existingIndex = @{}
        foreach ($item in $existingItems) {
            $existingIndex[$item.Title] = $item.Id
        }
    }

    process {
        $environmentToUse = $script:resolvedEnvironmentName
        Write-Verbose "Retrieving flows from environment '$environmentToUse'"
        $flowsJson = m365 flow list --environmentName $environmentToUse --output json --query '[].{DisplayName:displayName, Name:name, State:properties.state}' 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve flows. CLI output: $flowsJson"
        }

        try {
            $flows = if ([string]::IsNullOrWhiteSpace($flowsJson)) { @() } else { @($flowsJson | ConvertFrom-Json) }
        }
        catch {
            throw "Failed to parse flow listing. Error: $($_.Exception.Message)"
        }

        $summary.FlowsDiscovered += $flows.Count

        if (-not $Force -and -not $PSCmdlet.ShouldContinue("Update list '$ListTitle' at '$SiteUrl'?", "Confirm dashboard updates")) {
            Write-Warning "Operation cancelled by user."
            return
        }

        foreach ($flow in $flows) {
            if (-not $IncludeStoppedFlows -and $flow.State -ne 'Started') {
                $summary.SkippedStopped++
                continue
            }

            Write-Verbose "Fetching latest run for flow '$($flow.DisplayName)'"
            $runJson = m365 flow run list --flow $flow.Name --environmentName $environmentToUse --output json --query '[0].{Status:properties.status, StartTime:properties.startTime}' 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to list runs for '$($flow.DisplayName)'. CLI output: $runJson"
                $summary.Errors++
                continue
            }

            if ([string]::IsNullOrWhiteSpace($runJson)) {
                $summary.SkippedNoRuns++
                continue
            }

            try {
                $latestRun = $runJson | ConvertFrom-Json
            }
            catch {
                Write-Warning "Failed to parse runs for '$($flow.DisplayName)'. Error: $($_.Exception.Message)"
                $summary.Errors++
                continue
            }

            if (-not $latestRun) {
                $summary.SkippedNoRuns++
                continue
            }

            $summary.FlowsProcessed++

            $existingId = $existingIndex[$flow.DisplayName]
            $dashboardEntry = [pscustomobject]@{
                FlowName  = $flow.DisplayName
                Status    = $latestRun.Status
                StartTime = if ($latestRun.StartTime) { [DateTime]$latestRun.StartTime } else { $null }
                Operation = if ($existingId) { 'Update' } else { 'Add' }
            }

            $results.Add($dashboardEntry) | Out-Null

            $commonArgs = @(
                '--listTitle', $ListTitle,
                '--webUrl', $SiteUrl,
                '--Title', $flow.DisplayName,
                '--Status', $latestRun.Status,
                '--LastRunTime', ($dashboardEntry.StartTime ? $dashboardEntry.StartTime.ToString('s') : '')
            )

            if ($existingId) {
                if ($PSCmdlet.ShouldProcess($flow.DisplayName, 'Update dashboard entry')) {
                    $updateArgs = @('spo', 'listitem', 'set', '--id', $existingId) + $commonArgs + @('--output', 'json')
                    $updateOutput = m365 @updateArgs 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to update dashboard entry for '$($flow.DisplayName)'. CLI output: $updateOutput"
                        $summary.Errors++
                    }
                    else {
                        $summary.ItemsUpdated++
                    }
                }
            }
            else {
                if ($PSCmdlet.ShouldProcess($flow.DisplayName, 'Add dashboard entry')) {
                    $addArgs = @('spo', 'listitem', 'add', '--contentType', 'Item') + $commonArgs + @('--output', 'json')
                    $addOutput = m365 @addArgs 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to add dashboard entry for '$($flow.DisplayName)'. CLI output: $addOutput"
                        $summary.Errors++
                    }
                    else {
                        $summary.ItemsAdded++
                    }
                }
            }
        }
    }

    end {
        if ($ExportCsv -and $results.Count -gt 0) {
            $timestamp = (Get-Date).ToString('yyyyMMdd-HHmmss')
            $filePath = Join-Path -Path (Get-Location) -ChildPath "FlowDashboard-$timestamp.csv"
            Write-Verbose "Exporting dashboard data to '$filePath'"
            $results | Export-Csv -Path $filePath -NoTypeInformation -Encoding UTF8 -Force
        }

        Write-Host "Flow dashboard summary:" -ForegroundColor Cyan
        Write-Host ("- Flows discovered    : {0}" -f $summary.FlowsDiscovered)
        Write-Host ("- Flows processed     : {0}" -f $summary.FlowsProcessed)
        Write-Host ("- Skipped (stopped)   : {0}" -f $summary.SkippedStopped)
        Write-Host ("- Skipped (no runs)   : {0}" -f $summary.SkippedNoRuns)
        Write-Host ("- Added entries       : {0}" -f $summary.ItemsAdded)
        Write-Host ("- Updated entries     : {0}" -f $summary.ItemsUpdated)
        Write-Host ("- Errors              : {0}" -f $summary.Errors)

        if ($results.Count -gt 0) {
            $results
        }
    }
}

# Example usage
Update-FlowRunDashboard -SiteUrl "https://contoso.sharepoint.com/sites/Automation" -ListTitle "FlowDashboard" -Verbose -ExportCsv
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

## Contributors

| Author(s) |
|-----------|
| [Ryan Healy](https://github.com/Ryan365Apps)|
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/flow-search-flows-for-connection" aria-hidden="true" />
```

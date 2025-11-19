

# Archive inactive Teams

## Summary

This function, `Archive-PnPInactiveTeams`, gets a list of all the inactive Teams, based on the given number of days and archives them one by one.

![Example Screenshot](assets/example.png)

## Implementation

Save this script to a PSM1 module file, like `archive-inactiveTeams.psm1`. Then import the module file with `Import-Module`:

```powershell

Import-Module archive-inactiveTeams.psm1 -Verbose

```
The `-Verbose` switch lists the functions that are imported.

Once the module is imported the function `Archive-PnPInactiveTeams` will be loaded and ready to use.

# [PowerShell](#tab/ps)

```powershell

<# Script to archive inactive Teams
Author: Nico De Cleyre - @nicodecleyre
Blog: https://www.nicodecleyre.com
v1.0 - 8/6/2023

#>
Function Archive-PnPInactiveTeams {
    <#
.SYNOPSIS
Script to archive the inactive Teams

.Description
By inputting a designated timeframe for inactivity, the script automatically identifies Teams that have remained dormant beyond the specified period. These Teams are then archived

This solution requires an Azure App registration with the following permissions:
- Reports.Read.All
- TeamSettings.ReadWrite.All

.Parameter TenandId
The ID of your tenant

.PARAMETER ClientId
The ID of your Azure app registration

.PARAMETER ClientSecret
The secret of your Azure app registration

.PARAMETER InactiveDays
The minimum number of days that a Team must be active in order to be archived otherwise. Possible values: 7, 30, 90 or 180

.Example 
Archive-PnPInactiveTeams -TenandId "XXXXXX" -ClientId "XXXXXX" -ClientSecret "XXXXXX" -InactiveDays 30

.Example 
Archive-PnPInactiveTeams -TenandId "XXXXXX" -ClientId "XXXXXX" -ClientSecret "XXXXXX" -InactiveDays 180

#>
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $true)]
        $TenantId,
        [Parameter(Mandatory = $true)]
        $ClientId,
        [Parameter(Mandatory = $true)]
        $ClientSecret,
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        [ValidateSet("7", "30", "90", "180")]
        $InactiveDays
    )

    begin {
        #Log in to Microsoft Graph
        Write-Host "Connecting to Microsoft Graph" -ForegroundColor Yellow

        $uri = "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token"

        $body = @{
            client_id     = $ClientId
            client_secret = $ClientSecret
            grant_type    = "client_credentials"
            scope         = "https://graph.microsoft.com/.default"
        }

        $response = Invoke-RestMethod -Uri $uri -Method Post -Body $body
        $accessToken = $response.access_token
    }

    process {
        $today = Get-Date
        # Get the Teams Team activity detail
        $inactiveTeamsUri = "https://graph.microsoft.com/v1.0/reports/getTeamsTeamActivityDetail(period='D$InactiveDays')"

        $inactiveTeamsHeader = @{
            Authorization = "Bearer $accessToken"
        }

        $inactiveTeamsResponse = Invoke-RestMethod -Uri $inactiveTeamsUri -Method Get -Headers $inactiveTeamsHeader

        $teams = $inactiveTeamsResponse | ConvertFrom-Csv | Where-Object { $_.'Last Activity Date' -ne "" }

        foreach ($team in $teams) {
            $lastActivityDate = $team.'Last Activity Date'
            $timeSpan = New-TimeSpan -Start $lastActivityDate -End $today
            if ($timeSpan.Days -gt $InactiveDays) {
                $teamId = $team.'Team Id'
                $teamName = $team.'Team Name'
                Write-Host "Team $teamName ($teamId) is inactive since $($timeSpan.Days) days" -ForegroundColor DarkYellow

                $archiveTeamUri = "https://graph.microsoft.com/v1.0/teams/$teamId/archive"
                Invoke-RestMethod -Uri $archiveTeamUri -Method Post -Headers $inactiveTeamsHeader
                Write-Host "Team $teamName ($teamId) is archived" -ForegroundColor Green
            }
        }
    }

    end {
    }
}

```

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Invoke-ArchiveInactiveTeams {
    <#
    .SYNOPSIS
    Archives Microsoft Teams that have been inactive longer than the specified period.

    .DESCRIPTION
    Uses Microsoft Teams activity reports and administrative APIs to identify inactive teams and archive them.

    .PARAMETER InactivePeriod
    Report window to evaluate inactivity. Supported values: D7, D30, D90, D180.

    .PARAMETER Force
    Skips confirmation prompts and archives teams immediately.

    .PARAMETER SetSharePointReadOnly
    Also sets the associated SharePoint site to read-only when archiving the team.

    .EXAMPLE
    Invoke-ArchiveInactiveTeams -InactivePeriod D30 -WhatIf

    .EXAMPLE
    Invoke-ArchiveInactiveTeams -InactivePeriod D90 -Force -SetSharePointReadOnly
    #>
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter(Mandatory = $true, HelpMessage = 'Inactivity window. Allowed values: D7, D30, D90, D180.')]
        [ValidateSet('D7','D30','D90','D180')]
        [string]$InactivePeriod,

        [Parameter()]
        [switch]$Force,

        [Parameter()]
        [switch]$SetSharePointReadOnly
    )

    begin {
        $script:Summary = [ordered]@{
            Evaluated = 0
            Archived  = 0
            Skipped   = 0
            Failures  = 0
        }

        Write-Host 'Ensuring Microsoft 365 CLI authentication.' -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "CLI login failed. Output: $loginOutput"
        }

        Write-Host "Retrieving Teams user activity report for period $InactivePeriod." -ForegroundColor Cyan
        $reportOutput = m365 teams report useractivityuserdetail --period $InactivePeriod --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve Teams activity report. CLI output: $reportOutput"
        }

        try {
            $script:ReportUsers = $reportOutput | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse Teams activity report. $($_.Exception.Message)"
        }

        Write-Host 'Listing tenant teams.' -ForegroundColor Cyan
        $teamsOutput = m365 teams team list --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list Teams. CLI output: $teamsOutput"
        }

        try {
            $script:Teams = $teamsOutput | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse Teams list. $($_.Exception.Message)"
        }

        if (-not $script:Teams) {
            throw 'No Teams were returned by the CLI. Ensure the account has the necessary permissions.'
        }

        $script:InactiveThreshold = [int]::Parse($InactivePeriod.Substring(1))
    }

    process {
        Write-Host "Evaluating $($script:Teams.Count) teams for inactivity." -ForegroundColor Cyan
        foreach ($team in $script:Teams) {
            $script:Summary.Evaluated++

            if ($team.isArchived) {
                Write-Verbose "Team '$($team.displayName)' is already archived; skipping."
                $script:Summary.Skipped++
                continue
            }

            $teamActivity = $script:ReportUsers | Where-Object { $_.'Team Id' -eq $team.id }
            $lastActivity = $teamActivity | Sort-Object { $_.'Last Activity Date' } -Descending | Select-Object -First 1

            if ($lastActivity) {
                $lastActiveDate = [datetime]$lastActivity.'Last Activity Date'
                $daysInactive = (Get-Date) - $lastActiveDate
                if ($daysInactive.Days -lt $script:InactiveThreshold) {
                    Write-Verbose "Team '$($team.displayName)' had activity $($daysInactive.Days) days ago; skipping."
                    $script:Summary.Skipped++
                    continue
                }
            }
            else {
                Write-Verbose "Team '$($team.displayName)' has no activity records; treating as inactive."
            }

            $actionDescription = "Archive team '$($team.displayName)'"
            if ($SetSharePointReadOnly) {
                $actionDescription += ' and set SharePoint site read-only'
            }

            if (-not ($Force.IsPresent -or $PSCmdlet.ShouldProcess($team.displayName, $actionDescription))) {
                $script:Summary.Skipped++
                continue
            }

            Write-Host "Archiving team '$($team.displayName)' ($($team.id))." -ForegroundColor DarkYellow
            $archiveArgs = @(
                'teams','team','archive',
                '--id', $team.id
            )
            if ($SetSharePointReadOnly) {
                $archiveArgs += '--shouldSetSpoSiteReadOnlyForMembers'
            }

            $archiveOutput = m365 @archiveArgs 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Failures++
                Write-Warning "Failed to archive team '$($team.displayName)'. CLI output: $archiveOutput"
                continue
            }

            $script:Summary.Archived++
            Write-Host "Archived team '$($team.displayName)'." -ForegroundColor Green
        }
    }

    end {
        Write-Host "Teams evaluated: $($script:Summary.Evaluated)" -ForegroundColor Cyan
        Write-Host "Teams archived: $($script:Summary.Archived)" -ForegroundColor Green
        Write-Host "Teams skipped: $($script:Summary.Skipped)" -ForegroundColor Yellow
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
}

# example usage
Invoke-ArchiveInactiveTeams -InactivePeriod D30 -WhatIf
```

***

## Contributors

| Author(s) |
|-----------|
| [Nico De Cleyre](https://www.nicodecleyre.com)|
| Heinrich Krause |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/teams-archive-inactive-teams" aria-hidden="true" />

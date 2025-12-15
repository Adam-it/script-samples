

# Retrieving SharePoint Site URL for Teams Channels

## Summary

You may want to retrieve the SharePoint sites of private and shared channels for different reasons like to add to eDiscovery when running against "Specific Locations" or just for reporting purposes. A teams can have up to 30 Private Channels and unlimited shared channels up to the maximum of 1000 channels. This script can help to identify the SharePoint Urls associated to private and shared channels.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Get-TeamsChannelSharePointUrl {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $false, HelpMessage = "Filter teams by display name containing the provided text.")]
        [string]$TeamDisplayNameFilter,

        [Parameter(Mandatory = $false, HelpMessage = "Export results to the provided CSV file path.")]
        [string]$OutputCsvPath
    )

    begin {
        Write-Verbose 'Ensuring CLI for Microsoft 365 authentication'
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to sign in with CLI for Microsoft 365. CLI output: $loginOutput"
        }

        $script:Results = @()
    }

    process {
        Write-Verbose 'Retrieving teams list'
        if ($TeamDisplayNameFilter) {
            $filterValue = $TeamDisplayNameFilter.ToLowerInvariant().Replace("'", "\\'")
            $teamListOutput = m365 teams team list --output json --query "[?contains(to_lower(displayName), '$filterValue')]" 2>&1
        }
        else {
            $teamListOutput = m365 teams team list --output json 2>&1
        }
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve teams. CLI output: $teamListOutput"
        }

        $teams = if ([string]::IsNullOrWhiteSpace($teamListOutput)) { @() } else { $teamListOutput | ConvertFrom-Json }

        foreach ($team in $teams) {
            Write-Verbose "Processing team '$($team.displayName)'"
            $channelsOutput = m365 teams channel list --teamId $team.id --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve channels for team '$($team.displayName)'. CLI output: $channelsOutput"
                continue
            }
            $channels = if ([string]::IsNullOrWhiteSpace($channelsOutput)) { @() } else { $channelsOutput | ConvertFrom-Json }

            foreach ($channel in $channels) {
                $sharePointUrl = $null
                if ($channel.membershipType -eq 'shared' -or $channel.membershipType -eq 'private') {
                    $channelInfoOutput = m365 teams channel get --teamId $team.id --channelId $channel.id --output json 2>&1
                    if ($LASTEXITCODE -eq 0 -and -not [string]::IsNullOrWhiteSpace($channelInfoOutput)) {
                        $channelInfo = $channelInfoOutput | ConvertFrom-Json
                        $sharePointUrl = $channelInfo.channelFolderRelativeUrl
                    }
                }
                else {
                    $sharePointUrl = $team.webUrl
                }

                $script:Results += [pscustomobject][ordered]@{
                    TeamId         = $team.id
                    TeamName       = $team.displayName
                    ChannelId      = $channel.id
                    ChannelName    = $channel.displayName
                    ChannelType    = $channel.membershipType
                    SharePointUrl  = $sharePointUrl
                }
            }
        }
    }

    end {
        if ($OutputCsvPath) {
            Write-Verbose "Exporting results to $OutputCsvPath"
            $script:Results | Export-Csv -NoTypeInformation -Path $OutputCsvPath
        }

        $script:Results
    }
}

Get-TeamsChannelSharePointUrl -Verbose
```

# [PnP PowerShell](#tab/pnpps)

```powershell

param (
    [Parameter(Mandatory = $true)]
    [string] $domain ,
    [Parameter(Mandatory = $true)]
    [string] $teamName 
)

$adminSiteURL = "https://$domain-Admin.SharePoint.com"
Connect-PnPOnline -Url $adminSiteURL

$team = Get-PnPTeamsTeam -Identity  $teamName

$m365GroupId = $team.GroupId

Get-PnPTenantSite | Where-Object { $_.Template -eq 'TEAMCHANNEL#1' -and $_.RelatedGroupId -eq $m365GroupId  } | select Url,Template, Title

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Retrieving SharePoint Site URL for Teams Channels](https://reshmeeauckloo.com/posts/powershell-get-teams-channel-sharepoint-site/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/teams-get-channel-spo-urls" aria-hidden="true" />

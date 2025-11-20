

# List guests within Teams in a tenant

## Summary

List all guests in Microsoft Teams teams in the tenant and exports the results in a CSV.

PnP PowerShell script uses Microsoft Graph behind the scenes to get all teams and guest users. it requires an application/user that has been granted the Microsoft Graph API permission : Group.Read.All or Group.ReadWrite.All

# [MicrosoftTeams PowerShell](#tab/teamsps)
```powershell
Install-Module MicrosoftTeams
Connect-MicrosoftTeams
$teams = @()
$externalteams = @()
$teams = get-team
foreach ($team in $teams){
  $groupid = ($team.groupid)
  $users = (Get-TeamUser -GroupId $team.groupid | Where-Object {$_.Role -eq "Guest"})
  $extcount = ($users.count)
  foreach ($extuser in $users){
    $id = $team.groupid
    $teamext = ((Get-Team | Where-Object {$_.groupid -eq "$id"}).DisplayName).ToString()
    $ext = $extuser.User
    $externalteams += [pscustomobject]@{
      ExtUser   = $ext
      GroupID   = $id
      TeamName  = $teamext
	} 
  }
}
 if ($externalteams.Count -gt 0){
    Write-Host "Exporting the guest members in teams results.."
    $externalteams | Export-Csv -Path "GuestUsersFromTeams.csv" -NoTypeInformation
    Write-Host "Completed."
 }
 else{
    Write-host "there are no external user added to any team in your organization" -ForegroundColor yellow
 }
```
[!INCLUDE [More about Microsoft Teams PowerShell](../../docfx/includes/MORE-TEAMSPS.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell
#Connect as an application/user that has been granted Microsoft Graph API permissions : Group.Read.All or Group.ReadWrite.All
$siteUrl = "https://contoso-admin.sharepoint.com"
Connect-PnPOnline -Url $siteUrl -Interactive

$teams = @()
$externalteams = @()
$teams = Get-PnPTeamsTeam
foreach ($team in $teams)
{
  $groupid = $team.groupid
  $users = Get-PnPTeamsUser -Team $groupid -Role Guest
  $extcount = $users.count
  if($extcount -gt 0)
  {
    foreach ($extuser in $users)
    {
        $externalteams += [pscustomobject]@{
        ExtUser   = $extuser.UserPrincipalName
        GroupID   = $groupid
        TeamName  = $team.DisplayName
        } 
    }
  }
}
 if ($externalteams.Count -gt 0)
 {
    Write-Host "Exporting the guest members in teams results.."
    $externalteams | Export-Csv -Path "GuestUsersFromTeams.csv" -NoTypeInformation
    Write-Host "Completed."
 }
 else
 {
    Write-host "there are no external user added to any team in your organization" -ForegroundColor yellow
 }

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
function Get-TeamsGuestUsersWithCli {
    <#
    .SYNOPSIS
    Exports all guest members across Microsoft Teams teams using CLI for Microsoft 365.

    .DESCRIPTION
    Ensures CLI authentication, retrieves all tenant teams, and enumerates guest members. Results are exported
    to a CSV for further review. Teams without guest members are skipped unless -IncludeEmptyTeams is supplied.

    .PARAMETER OutputPath
    Destination CSV file path. Defaults to ./GuestUsersFromTeams.csv.

    .PARAMETER IncludeEmptyTeams
    Include teams with zero guest members in the output.

    .EXAMPLE
    Get-TeamsGuestUsersWithCli -OutputPath ./GuestUsersFromTeams.csv -Verbose
    #>
    [CmdletBinding()]
    param(
        [Parameter()]
        [ValidateNotNullOrEmpty()]
        [string]$OutputPath = './GuestUsersFromTeams.csv',

        [Parameter()]
        [switch]$IncludeEmptyTeams
    )

    begin {
        Write-Host 'Ensuring Microsoft 365 CLI authentication.' -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "CLI login failed. Output: $loginOutput"
        }

        $script:Guests = @()
        $script:Summary = [ordered]@{
            TeamsProcessed   = 0
            TeamsWithGuests  = 0
        }
    }

    process {
        Write-Host 'Retrieving tenant teams.' -ForegroundColor Cyan
        $teamsJson = m365 teams team list --output json --query "[].{id:id,displayName:displayName}" 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Unable to list teams. CLI output: $teamsJson"
        }

        try {
            $teams = $teamsJson | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse teams list. $($_.Exception.Message)"
        }

        if (-not $teams) {
            Write-Warning 'No teams were returned by the CLI.'
            return
        }

        foreach ($team in $teams) {
            $script:Summary.TeamsProcessed++
            Write-Verbose "Processing team '$($team.displayName)' ($($team.id))."

            $guestsJson = m365 teams user list --teamId $team.id --role Guest --output json --query "[].userPrincipalName" 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve guests for team '$($team.displayName)'. CLI output: $guestsJson"
                continue
            }

            $guestUpns = @()
            if (-not [string]::IsNullOrWhiteSpace($guestsJson)) {
                try {
                    $guestUpns = $guestsJson | ConvertFrom-Json -ErrorAction Stop
                }
                catch {
                    Write-Warning "Unable to parse guest list for team '$($team.displayName)'. $($_.Exception.Message)"
                    continue
                }
            }

            if ($guestUpns.Count -gt 0) {
                $script:Summary.TeamsWithGuests++
                foreach ($upn in $guestUpns) {
                    $script:Guests += [pscustomobject]@{
                        TeamId    = $team.id
                        TeamName  = $team.displayName
                        GuestUser = $upn
                    }
                }
            }
            elseif ($IncludeEmptyTeams.IsPresent) {
                $script:Guests += [pscustomobject]@{
                    TeamId    = $team.id
                    TeamName  = $team.displayName
                    GuestUser = ''
                }
            }
        }
    }

    end {
        if ($script:Guests.Count -eq 0) {
            Write-Host 'No guest users found across the processed teams.' -ForegroundColor Yellow
            return
        }

        $directory = Split-Path -Path $OutputPath -Parent
        if ([string]::IsNullOrEmpty($directory)) {
            $directory = '.'
        }
        if (-not (Test-Path -Path $directory -PathType Container)) {
            throw "Output directory '$directory' does not exist."
        }

        $script:Guests | Export-Csv -Path $OutputPath -Encoding UTF8 -NoTypeInformation
        Write-Host "Guest report saved to '$OutputPath'." -ForegroundColor Green

        Write-Host "Teams processed: $($script:Summary.TeamsProcessed)" -ForegroundColor Cyan
        Write-Host "Teams with guests: $($script:Summary.TeamsWithGuests)" -ForegroundColor Cyan
        Write-Host "Guest rows exported: $($script:Guests.Count)" -ForegroundColor Cyan
    }
}

# example usage
Get-TeamsGuestUsersWithCli -OutputPath ./GuestUsersFromTeams.csv -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Jiten Parmar](https://github.com/jitenparmar) |
| [Leon Armston](https://github.com/LeonArmston) |
| [Jasey Waegebaert](https://github.com/Jwaegebaert) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/teams-list-guestusers" aria-hidden="true" />

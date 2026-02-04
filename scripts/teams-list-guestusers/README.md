

# List guests within Teams in a tenant

## Summary

List all guests in Microsoft Teams teams in the tenant and exports the results to a CSV file. The script supports PnP PowerShell, Microsoft Teams PowerShell, and CLI for Microsoft 365.

The PnP PowerShell and CLI for Microsoft 365 scripts use Microsoft Graph behind the scenes to get all teams and guest users. They require an application/user that has been granted the Microsoft Graph API permissions: Group.Read.All or Group.ReadWrite.All.

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
[CmdletBinding()]
param(
    [Parameter(Mandatory = $false, HelpMessage = "Path where the CSV report will be saved")]
    [ValidateScript({
        if (Test-Path -Path $_ -PathType Container) { $true }
        else { throw "Path '$_' does not exist or is not a directory" }
    })]
    [string]$OutputPath = (Get-Location).Path,

    [Parameter(Mandatory = $false, HelpMessage = "Minimum number of guests a team must have to be included in the report (default: 1)")]
    [ValidateRange(1, [int]::MaxValue)]
    [int]$MinimumGuestCount = 1
)

begin {
    Write-Verbose "Starting Microsoft Teams guest user report generation"
    Write-Verbose "Output path: $OutputPath"
    Write-Verbose "Minimum guest count filter: $MinimumGuestCount"

    $transcriptPath = Join-Path -Path $OutputPath -ChildPath "TeamsGuestUsers_Transcript_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    Start-Transcript -Path $transcriptPath

    Write-Verbose "Ensuring user is signed in to Microsoft 365"
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate to Microsoft 365. Please check your credentials and try again."
    }
    Write-Verbose "Successfully authenticated to Microsoft 365"

    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath -PathType Container)) {
            throw "Output path '$OutputPath' does not exist or is not a directory"
        }
    }

    $script:ReportCollection = [System.Collections.ArrayList]@()
    $script:Summary = @{
        TotalTeams = 0
        TeamsWithGuests = 0
        TotalGuests = 0
        Failures = 0
    }
}

process {
    Write-Verbose "Retrieving all Microsoft Teams in the tenant"
    $teamsJson = m365 teams team list --output json
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve Microsoft Teams. Error code: $LASTEXITCODE"
    }

    $teams = @($teamsJson | ConvertFrom-Json)
    $script:Summary.TotalTeams = $teams.Count
    Write-Verbose "Found $($teams.Count) teams in the tenant"

    if ($teams.Count -eq 0) {
        Write-Warning "No teams found in the tenant"
        return
    }

    $teamCounter = 0
    foreach ($team in $teams) {
        $teamCounter++
        Write-Verbose "Processing team $teamCounter of $($teams.Count): $($team.displayName)"

        try {
            $usersJson = m365 teams user list --teamId $team.id --role Guest --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve guest users for team '$($team.displayName)' (ID: $($team.id))"
                $script:Summary.Failures++
                continue
            }

            $guestUsers = @($usersJson | ConvertFrom-Json)
            $guestCount = $guestUsers.Count

            if ($guestCount -ge $MinimumGuestCount) {
                Write-Verbose "Team '$($team.displayName)' has $guestCount guest user(s)"
                $script:Summary.TeamsWithGuests++

                foreach ($guest in $guestUsers) {
                    $reportItem = [PSCustomObject]@{
                        TeamName = $team.displayName
                        TeamId = $team.id
                        GuestUserPrincipalName = $guest.userPrincipalName
                        GuestDisplayName = $guest.displayName
                        GuestId = $guest.id
                    }
                    [void]$script:ReportCollection.Add($reportItem)
                    $script:Summary.TotalGuests++
                }
            }
            else {
                Write-Verbose "Team '$($team.displayName)' has $guestCount guest user(s) - below minimum threshold of $MinimumGuestCount"
            }
        }
        catch {
            Write-Warning "Error processing team '$($team.displayName)': $($_.Exception.Message)"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Verbose "Generating CSV report"

    $csvFileName = "TeamsGuestUsers_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
    $csvPath = Join-Path -Path $OutputPath -ChildPath $csvFileName

    if ($script:ReportCollection.Count -gt 0) {
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "`n========================================" -ForegroundColor Cyan
        Write-Host "Microsoft Teams Guest User Report" -ForegroundColor Cyan
        Write-Host "========================================" -ForegroundColor Cyan
        Write-Host "Total teams processed:      " -NoNewline; Write-Host $script:Summary.TotalTeams -ForegroundColor Green
        Write-Host "Teams with guests:          " -NoNewline; Write-Host $script:Summary.TeamsWithGuests -ForegroundColor Green
        Write-Host "Total guest users found:    " -NoNewline; Write-Host $script:Summary.TotalGuests -ForegroundColor Green
        if ($script:Summary.Failures -gt 0) {
            Write-Host "Failed team retrievals:     " -NoNewline; Write-Host $script:Summary.Failures -ForegroundColor Red
        }
        else {
            Write-Host "Failed team retrievals:     " -NoNewline; Write-Host $script:Summary.Failures -ForegroundColor Green
        }
        Write-Host "`nReport exported to: " -NoNewline; Write-Host $csvPath -ForegroundColor Yellow
        Write-Host "========================================`n" -ForegroundColor Cyan
    }
    else {
        Write-Host "`n========================================" -ForegroundColor Yellow
        Write-Host "Microsoft Teams Guest User Report" -ForegroundColor Yellow
        Write-Host "========================================" -ForegroundColor Yellow
        Write-Host "No guest users found in any team" -ForegroundColor Yellow
        Write-Host "Total teams processed: $($script:Summary.TotalTeams)" -ForegroundColor Yellow
        if ($script:Summary.Failures -gt 0) {
            Write-Host "Failed team retrievals: $($script:Summary.Failures)" -ForegroundColor Red
        }
        Write-Host "========================================`n" -ForegroundColor Yellow
    }

    Stop-Transcript
    Write-Verbose "Transcript saved to: $transcriptPath"
}

# Usage examples:
#
# Example 1: Basic usage - export all guest users to current directory
# .\.teams-list-guestusers.ps1
#
# Example 2: Export to specific directory
# .\.teams-list-guestusers.ps1 -OutputPath "C:\Reports"
#
# Example 3: Filter teams with at least 5 guests
# .\.teams-list-guestusers.ps1 -MinimumGuestCount 5
#
# Example 4: Verbose output with custom path
# .\.teams-list-guestusers.ps1 -OutputPath "C:\Reports" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Jiten Parmar](https://github.com/jitenparmar) |
| [Leon Armston](https://github.com/LeonArmston) |
| [Jasey Waegebaert](https://github.com/Jwaegebaert) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/teams-list-guestusers" aria-hidden="true" />

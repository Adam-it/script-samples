

# List Ownerless Teams

## Summary

If your organization has been using Microsoft Teams for more that a few years, you'll no doubt have a number of Teams that have been orphaned, probably because the original owner has moved on. This script will list all Teams that have fewer than a specified number of owners, and export them to a CSV file.

# [PnP PowerShell](#tab/pnpps)

```powershell

Connect-PnPOnline -ClientId "GUID" -Tenant "GUID" -CertificatePath -CertificatePassword "Password" 

$MinimumRequiredOwners = 1;

$Groups = Get-PnPMicrosoft365Group -IncludeOwners | Where-Object {$_.Owners.Count -le $MinimumRequiredOwners -and $_.HasTeam}

$MappedObjects = [System.Collections.ArrayList]@()

foreach ($Group in $Groups) {
    $MappedObject = [PSCustomObject]@{
        GroupId = $Group.GroupId
        DisplayName = $Group.DisplayName
        OwnersCount = $Group.Owners.Count
        Owners = $Group.Owners | Select-Object -Property "Email" | Join-String -Property "Email" -Separator "; "
    }
    $MappedObjects += $MappedObject
}

$fileName = "$(Get-Date -Format ("yyyy-MM-dd"))-GroupsWithFewerThan$($MinimumRequiredOwners)Owners.csv";
$MappedObjects | Select-Object -Property * | Export-Csv -Path ".\$fileName" -Encoding UTF8 -Delimiter ";" -Force;

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Get-OwnerlessTeamsWithCli {
    <#
    .SYNOPSIS
    Lists Microsoft Teams teams with fewer owners than the specified threshold using CLI for Microsoft 365.

    .DESCRIPTION
    Retrieves tenant teams, inspects their owners, and exports a CSV report containing those with owner
    counts less than or equal to the configured threshold. Report naming mirrors the PnP variant.

    .PARAMETER MinimumOwners
    Minimum number of owners a team should have. Teams with owner count less than or equal to this value are reported.

    .PARAMETER OutputDirectory
    Directory where the CSV report will be written. Defaults to the current directory.

    .EXAMPLE
    Get-OwnerlessTeamsWithCli -MinimumOwners 1 -OutputDirectory ./reports
    #>
    [CmdletBinding()]
    param(
        [Parameter()]
        [ValidateRange(0,10)]
        [int]$MinimumOwners = 1,

        [Parameter()]
        [ValidateNotNullOrEmpty()]
        [string]$OutputDirectory = '.'
    )

    begin {
        if (-not (Test-Path -Path $OutputDirectory -PathType Container)) {
            throw "Output directory '$OutputDirectory' does not exist."
        }

        $script:OutputDirectory = (Resolve-Path -Path $OutputDirectory).Path
        $timestamp = Get-Date -Format 'yyyy-MM-dd'
        $script:ReportPath = Join-Path $script:OutputDirectory "${timestamp}-GroupsWithFewerThan${MinimumOwners}Owners.csv"
        $script:Results = @()

        Write-Host 'Ensuring CLI authentication.' -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "CLI login failed. Output: $loginOutput"
        }
    }

    process {
        Write-Host 'Retrieving Microsoft Teams teams.' -ForegroundColor Cyan
        $teamsJson = m365 teams team list --output json --query "[].{id:id,displayName:displayName}" 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Unable to list teams. CLI output: $teamsJson"
        }

        try {
            $teams = $teamsJson | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse team list. $($_.Exception.Message)"
        }

        if (-not $teams) {
            Write-Warning 'No teams returned by CLI. Ensure the account has sufficient permissions.'
            return
        }

        foreach ($team in $teams) {
            Write-Verbose "Evaluating team '$($team.displayName)' ($($team.id))."

            $ownersJson = m365 teams user list --teamId $team.id --role Owner --output json --query "[].userPrincipalName" 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve owners for team '$($team.displayName)'. CLI output: $ownersJson"
                continue
            }

            $owners = @()
            if (-not [string]::IsNullOrWhiteSpace($ownersJson)) {
                try {
                    $owners = $ownersJson | ConvertFrom-Json -ErrorAction Stop
                }
                catch {
                    Write-Warning "Unable to parse owner list for team '$($team.displayName)'. $($_.Exception.Message)"
                    continue
                }
            }

            $ownerCount = if ($owners) { $owners.Count } else { 0 }
            if ($ownerCount -le $MinimumOwners) {
                $script:Results += [pscustomobject]@{
                    TeamId      = $team.id
                    DisplayName = $team.displayName
                    OwnersCount = $ownerCount
                    Owners      = if ($owners) { ($owners -join '; ') } else { '' }
                }
            }
        }
    }

    end {
        if ($script:Results.Count -gt 0) {
            $script:Results | Export-Csv -Path $script:ReportPath -Encoding UTF8 -NoTypeInformation
            Write-Host "Report saved to '$($script:ReportPath)'." -ForegroundColor Green
        }
        else {
            Write-Host 'No teams matched the owner threshold. No report generated.' -ForegroundColor Yellow
        }

        Write-Host "Teams reported: $($script:Results.Count)" -ForegroundColor Green
    }
}

# example usage
Get-OwnerlessTeamsWithCli -MinimumOwners 1 -OutputDirectory ./reports
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Dan Toft](https://Dan-Toft.dk) |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/teams-list-ownerless-teams" aria-hidden="true" />



# Identifying Duplicate Microsoft 365 Group Names

## Summary

It is possible to create M365 Groups and Teams with the same name, and there is currently no built-in way to prevent this. Having duplicate names can cause confusion and increase security, governance and compliance risks.

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
$dateTime = (Get-Date).toString("dd-MM-yyyy-hh-ss")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = (Split-Path $invocation.MyCommand.Path) + "\"
$exportFilePath = Join-Path -Path $directorypath -ChildPath $([string]::Concat($domain,"-duplicateM365_",$dateTime,".csv"));

$adminSiteURL = "https://$domain-Admin.SharePoint.com"
Connect-PnPOnline -Url $adminSiteURL

# Retrieve all M365 groups
$groups = get-PnPMicrosoft365Group

# Find duplicate group names
$duplicateGroups = $groups | Group-Object DisplayName | Where-Object { $_.Count -gt 1 }

# Create a report
$report = @()
foreach ($group in $duplicateGroups) {
    foreach ($item in $group.Group) {
        $report += [PSCustomObject]@{
            DisplayName = $item.DisplayName
            GroupId     = $item.Id
            Mail        = $item.Mail
        }
    }
}

# Export the report to a CSV file
$report | Export-Csv -Path $exportFilePath -NoTypeInformation
Disconnect-PnPOnline
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]

begin {
    m365 login --ensure

    $script:Summary = [ordered]@{
        GroupsEvaluated   = 0
        DuplicateClusters = 0
    }

    $script:Duplicates = [System.Collections.Generic.List[psobject]]::new()
    Write-Host "Fetching Microsoft 365 groups..."
}

process {
    $groupsJson = m365 entra m365group list --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve Microsoft 365 groups. CLI output: $groupsJson"
    }

    $groups = if ([string]::IsNullOrWhiteSpace($groupsJson)) { @() } else { $groupsJson | ConvertFrom-Json }
    $Summary.GroupsEvaluated = $groups.Count

    $duplicates = $groups | Group-Object displayName | Where-Object { $_.Count -gt 1 }
    foreach ($cluster in $duplicates) {
        $Summary.DuplicateClusters++
        foreach ($item in $cluster.Group) {
            $Duplicates.Add([pscustomobject]@{
                DisplayName = $item.displayName
                GroupId     = $item.id
                Mail        = $item.mail
            })
        }
    }
}

end {
    if ($Duplicates.Count -eq 0) {
        Write-Host "No duplicate Microsoft 365 group display names found."
        Write-Host ("Summary: {0} groups evaluated." -f $Summary.GroupsEvaluated)
        return
    }

    Write-Host "Duplicate Microsoft 365 group names detected (grouped by display name):"
    $Duplicates | Group-Object DisplayName | ForEach-Object {
        Write-Host "  DisplayName: $($_.Name)"
        $_.Group | ForEach-Object {
            Write-Host ("    Id: {0} Mail: {1}" -f $_.GroupId, $_.Mail)
        }
    }

    Write-Host ("Summary: {0} groups evaluated, {1} duplicate clusters found, {2} duplicate entries listed." -f `
        $Summary.GroupsEvaluated, $Summary.DuplicateClusters, $Duplicates.Count)
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

Sample first appeared on [Identifying Duplicate Microsoft 365 Group Names](https://reshmeeauckloo.com/posts/powershell-duplicate-m365group-teams/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| Adamm Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/aad-get-duplicate-m365group" aria-hidden="true" />

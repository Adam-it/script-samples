

# Replace a user's membership in selected Microsoft 365 Groups or Teams

## Summary

This script can be used to replace the membership of a user for a selected list of Groups. It might be useful when a person changes role in an organization or is about to leave it.
 
[!INCLUDE [Delete Warning](../../docfx/includes/DELETE-WARN.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell
$adminUrl = "https://contoso-admin.sharepoint.com"
$fileInput = "<PUTYOURPATHHERE.csv>"
$oldUser = "upnOfOldUser"
$newUser = "upnOfNewUser"

Connect-PnPOnline -Url $adminUrl -Interactive

function Replace-Membership {
  [cmdletbinding()]
  param(
    [parameter(Mandatory = $true)]
    $fileInput ,
    [parameter(Mandatory = $true)]
    $oldUser,
    [parameter(Mandatory = $true)]
    $newUser
  )
  $groupsToProcess = Import-Csv $fileInput 
  $groupsToProcess.id | ForEach-Object {
    $groupId = $_
    Write-Host "Processing Group ($groupId)" -ForegroundColor DarkGray -NoNewline

    $group = $null
    try {
      $group = Get-PnPMicrosoft365Group -Identity $groupId 
    }
    catch {
      Write-Host
      Write-Host $_.Exception.Message -ForegroundColor Red
      return
    }
    Write-Host " - $($group.displayName)" -ForegroundColor DarkGray

    $isTeam = $group.resourceProvisioningOptions.Contains("Team");

    $users = $null
    $members = Get-PnPMicrosoft365GroupMembers -Identity $groupId
    $owners = Get-PnPMicrosoft365GroupOwners -Identity $groupId
    $members | Where-Object { $_.userPrincipalName -eq $oldUser } | ForEach-Object {
      $user = $_
      $owner = $owners |  Where-Object { $_.userPrincipalName -eq $oldUser }
      if($owner)
      {
        Write-Host "Found $oldUser with owner rights" -ForegroundColor Green
      }
      else
      {
        Write-Host "Found $oldUser with member rights" -ForegroundColor Green
      }
      
      # owners must be explicitly added as members if it is a team
      if ($user -or $isTeam) {
        try {
          Write-Host "Granting $newUser member rights"
          Add-PnPMicrosoft365GroupMember -Identity $groupId -Users $newUser
        }
        catch {
          Write-Host $_.Exception.Message -ForegroundColor White
        } 
      }

      if ($owner) {
        try {
          Write-Host "Granting $newUser owner rights"
          Add-PnPMicrosoft365GroupOwner -Identity $groupId -Users $newUser
        }
        catch {
          Write-Host $_.Exception.Message -ForegroundColor White
        }
      }

      try {
        Write-Host "Removing $oldUser..."
        if($owner)
        {
        Remove-PnPMicrosoft365GroupOwner -Identity $groupId -Users $oldUser 
        }
        Remove-PnPMicrosoft365GroupMember -Identity $groupId -Users $oldUser
      }
      catch {
        Write-Host $_.Exception.Message -ForegroundColor Red
        continue
      }
    }
  }
}

Replace-Membership $fileInput $oldUser $newUser
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory, HelpMessage = "Path to CSV file containing group IDs in an 'id' column.")]
    [string]$GroupsCsvPath,

    [Parameter(Mandatory, HelpMessage = "UPN of the user whose membership should be replaced.")]
    [string]$OldUserUpn,

    [Parameter(Mandatory, HelpMessage = "UPN of the user who will replace the old user.")]
    [string]$NewUserUpn,

    [switch]$Force
)

begin {
    if (-not (Test-Path -Path $GroupsCsvPath -PathType Leaf)) {
        throw "CSV file not found at path: $GroupsCsvPath"
    }

    m365 login --ensure

    $script:Groups = Import-Csv -Path $GroupsCsvPath
    if (-not $Groups) {
        throw "No group IDs found in CSV. Ensure the file contains an 'id' column."
    }

    $script:Summary = [ordered]@{
        GroupsProcessed = 0
        MembersAdded    = 0
        OwnersAdded     = 0
        UsersRemoved    = 0
        Failures        = 0
    }

    Write-Host "Processing membership replacement for $($Groups.Count) group(s)."
}

process {
    foreach ($group in $Groups) {
        $groupId = $group.id
        if (-not $groupId) {
            Write-Warning "Skipping entry without 'id' value."
            continue
        }

        $Summary.GroupsProcessed++
        Write-Host "\nGroup: $groupId"

        $owners = m365 entra m365group user list --groupId $groupId --role Owner --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "  Failed to retrieve owners. CLI: $owners"
            $Summary.Failures++
            continue
        }
        $owners = if ([string]::IsNullOrWhiteSpace($owners)) { @() } else { $owners | ConvertFrom-Json }

        $members = m365 entra m365group user list --groupId $groupId --role Member --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "  Failed to retrieve members. CLI: $members"
            $Summary.Failures++
            continue
        }
        $members = if ([string]::IsNullOrWhiteSpace($members)) { @() } else { $members | ConvertFrom-Json }

        $oldIsOwner = $owners | Where-Object { $_.userPrincipalName -eq $OldUserUpn }
        $oldIsMember = $members | Where-Object { $_.userPrincipalName -eq $OldUserUpn }

        if (-not ($oldIsOwner -or $oldIsMember)) {
            Write-Host "  $OldUserUpn is not part of this group; skipping."
            continue
        }

        if (-not $PSCmdlet.ShouldProcess($groupId, "Replace membership of $OldUserUpn with $NewUserUpn")) {
            Write-Host "  WhatIf: changes skipped."
            continue
        }

        try {
            if ($oldIsMember -or $oldIsOwner) {
                Write-Host "  Ensuring $NewUserUpn is a member"
                $addMember = m365 entra m365group user add --groupId $groupId --userNames $NewUserUpn --role Member --output json 2>&1
                if ($LASTEXITCODE -eq 0) {
                    $Summary.MembersAdded++
                } elseif ($addMember -notmatch 'is already a member') {
                    throw "Failed to add member. CLI: $addMember"
                }
            }

            if ($oldIsOwner) {
                Write-Host "  Ensuring $NewUserUpn is an owner"
                $addOwner = m365 entra m365group user set --groupId $groupId --userNames $NewUserUpn --role Owner --output json 2>&1
                if ($LASTEXITCODE -eq 0) {
                    $Summary.OwnersAdded++
                } elseif ($addOwner -notmatch 'already has the role Owner') {
                    throw "Failed to add owner. CLI: $addOwner"
                }
            }

            Write-Host "  Removing $OldUserUpn"
            $removeArgs = @('entra','m365group','user','remove','--groupId',$groupId,'--userNames',$OldUserUpn,'--output','json')
            if ($Force) { $removeArgs += '--force' }
            $removeResult = m365 @removeArgs 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to remove user. CLI: $removeResult"
            }
            $Summary.UsersRemoved++
        }
        catch {
            Write-Warning "  $_"
            $Summary.Failures++
        }
    }
}

end {
    Write-Host "\nSummary:"
    Write-Host ("  Groups processed: {0}" -f $Summary.GroupsProcessed)
    Write-Host ("  Members ensured: {0}" -f $Summary.MembersAdded)
    Write-Host ("  Owners ensured: {0}" -f $Summary.OwnersAdded)
    Write-Host ("  Old user removals: {0}" -f $Summary.UsersRemoved)
    Write-Host ("  Failures: {0}" -f $Summary.Failures)
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***


**CSV Format**
```csv
id
<groupId>
<groupId>
<groupId>
```


## Contributors

| Author(s) |
|-----------|
| Reshmee Auckloo |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/aad-replace-membership-of-selected-groups" aria-hidden="true" />

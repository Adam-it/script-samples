# Delete all Microsoft 365 groups and SharePoint sites

## Summary

This script sample shows how you can delete Microsoft 365 Groups and associated SharePoint Online sites in your development environment.
 
[!INCLUDE [Delete Warning](../../docfx/includes/DELETE-WARN.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
$AdminCenterURL="https://contoso-admin.sharepoint.com/"

#Connect to SharePoint admin URL using PnPOnline to use PnP cmdlets to delete M365 groups and SharePoint sites
Connect-PnPOnline -Url $AdminCenterURL -Interactive

#Retrieve all M365 group connected (template "GROUP#0" sites to be deleted) sites beginning with https://contoso.sharepoint.com/sites/D-Test
$sites = Get-PnPTenantSite -Filter {Url -like https://contoso.sharepoint.com/sites/D-Test} -Template 'GROUP#0'

#Displaying the sites returned to be deleted
$sites | Format-Table  Url, Template, GroupId

Read-Host -Prompt "Press Enter to start deleting m365 groups and sites (CTRL + C to exit)"

$sites | ForEach-Object {
    #Delete M365 group
    Remove-PnPMicrosoft365Group -Identity $_.GroupId

    #Allow time for M365 group to be deleted
    Start-Sleep -Seconds 60

    #Delete the SharePoint site after the M365 group is deleted
    Remove-PnPTenantSite -Url $_.Url -Force -SkipRecycleBin

    #Permanently remove the M365 group
    Remove-PnPDeletedMicrosoft365Group -Identity $_.GroupId

    #Permanently delete the site and to allow a site to be created with the same URL of the site just deleted, i.e. to avoid message "This site address is available with modification"
    Remove-PnPTenantDeletedSite -Identity $_.Url -Force
}

# Disconnect SharePoint online connection
Disconnect-PnPOnline
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $false, ValueFromPipeline, ValueFromPipelineByPropertyName, HelpMessage = "Display name filter used to select Microsoft 365 groups for deletion.")]
    [string[]]$DisplayNameFilter = @("Permission"),
    [Parameter(Mandatory = $false, HelpMessage = "Skip interactive confirmation before deleting groups.")]
    [switch]$Force,
    [Parameter(Mandatory = $false, HelpMessage = "Permanently delete groups without placing them in the recycle bin.")]
    [switch]$HardDelete
)

begin {
    try {
        m365 login --ensure
    }
    catch {
        Write-Error "Unable to establish a CLI for Microsoft 365 session. $_"
        return
    }

    $script:totalMatched = 0
    $script:totalDeleted = 0
    $script:totalFailed = 0
}

process {
    foreach ($filter in $DisplayNameFilter) {
        $groups = m365 entra m365group list --displayName "$filter" --output json | ConvertFrom-Json
        $groups = @($groups)

        if (-not $groups) {
            Write-Warning "No Microsoft 365 groups found matching display name filter: $filter"
            continue
        }

        $script:totalMatched += $groups.Count

        Write-Host "Microsoft 365 groups scheduled for deletion using filter '$filter':"
        $groups | Format-Table displayName, id, mail

        if ($HardDelete.IsPresent) {
            Write-Warning "Hard delete is enabled. Groups will be permanently removed and cannot be recovered."
        }

        if (-not $Force.IsPresent) {
            Read-Host -Prompt "Press Enter to start deleting M365 groups and associated SharePoint sites for filter '$filter' (CTRL + C to exit)"
        }

        $groups | ForEach-Object {
            $removeArgs = @('entra', 'm365group', 'remove', '--id', $_.id, '--force')
            if ($HardDelete.IsPresent) {
                $removeArgs += '--skipRecycleBin'
                Write-Host "Permanently deleting M365 group: $($_.displayName)"
            }
            else {
                Write-Host "Soft deleting M365 group (moved to recycle bin): $($_.displayName)"
            }

            & m365 @removeArgs
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to delete M365 group: $($_.displayName) ($($_.id)). Exit code: $LASTEXITCODE"
                $script:totalFailed++
                continue
            }
            Write-Host "Deleted M365 group: $($_.displayName)"
            $script:totalDeleted++
        }
    }
}

end {
    Write-Host "Deletion summary:"
    Write-Host (" - Matched groups: {0}" -f $script:totalMatched)
    Write-Host (" - Successfully deleted: {0}" -f $script:totalDeleted)
    Write-Host (" - Failed deletions: {0}" -f $script:totalFailed)
}

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Reshmee Auckloo |
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/) |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/aad-delete-m365-groups-and-sharepoint-sites" aria-hidden="true" />

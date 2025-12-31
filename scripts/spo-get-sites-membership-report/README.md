

# Get membership report of site(s) within tenant

## Summary

The scripts get membership permission report of site(s) within tenant and export it to a CSV file. The report retrieves
-  Site Admins
-  m365 Group Owners
-  m365 Group Members
-  m365 Group Guests
-  Site Owners
-  Site Members
-  Site Visitors

![PnP Powershell result](assets/preview.png)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage = "SharePoint Admin Center URL (e.g., https://contoso-admin.sharepoint.com)")]
    [string]$TenantAdminUrl,

    [Parameter(HelpMessage = "OData filter for sites (default: only /sites/ paths)")]
    [string]$SiteFilter = "Url -like '/sites/' and Template ne 'RedirectSite#0'",

    [Parameter(HelpMessage = "Path for CSV export (default: timestamped file in current directory)")]
    [string]$OutputPath
)

begin {
    $timestamp = (Get-Date).ToString("yyyyMMdd_HHmmss")
    if (-not $OutputPath) {
        $OutputPath = Join-Path (Get-Location) "SitesMembershipReport_$timestamp.csv"
    }

    $logFile = Join-Path (Get-Location) "SitesMembershipReport_$timestamp.log"
    Start-Transcript -Path $logFile | Out-Null

    Write-Host "Starting Sites Membership Report..." -ForegroundColor Cyan
    Write-Host "Tenant Admin URL: $TenantAdminUrl" -ForegroundColor White
    Write-Host "Site Filter: $SiteFilter" -ForegroundColor White
    Write-Host "Output Path: $OutputPath" -ForegroundColor White
    Write-Host "`n"

    $script:Summary = @{
        SitesScanned = 0
        Successes = 0
        Failures = 0
    }

    $script:ReportCollection = [System.Collections.Generic.List[PSCustomObject]]::new()

    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please check your credentials."
    }
    Write-Verbose "Login successful"
}

process {
    try {
        Write-Host "Retrieving sites from tenant..." -ForegroundColor Yellow
        Write-Verbose "Executing: m365 spo site list --filter '$SiteFilter' --output json"
        $sitesJson = m365 spo site list --filter $SiteFilter --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sites: $sitesJson"
        }

        $sites = $sitesJson | ConvertFrom-Json
        Write-Host "Found $($sites.Count) site(s) matching filter" -ForegroundColor Green
        Write-Host "`n"

        foreach ($site in $sites) {
            $script:Summary.SitesScanned++
            Write-Host "[$($script:Summary.SitesScanned)/$($sites.Count)] Processing: $($site.Title)" -ForegroundColor Cyan
            Write-Verbose "  Site URL: $($site.Url)"

            try {
                $reportItem = [PSCustomObject]@{
                    'Site Name' = $site.Title
                    'Group Owners' = ''
                    'Group Members' = ''
                    'Group Guests' = ''
                    'Site Id' = $site.SiteId
                    'Site admins' = ''
                    'Site owners' = ''
                    'Site members' = ''
                    'Site visitors' = ''
                }

                Write-Verbose "  Retrieving site admins..."
                $adminJson = m365 spo site admin list --siteUrl $site.Url --output json 2>&1
                if ($LASTEXITCODE -eq 0) {
                    $admins = $adminJson | ConvertFrom-Json
                    $reportItem.'Site admins' = ($admins.Title | Where-Object { $_ }) -join ';'
                    Write-Verbose "    Found $($admins.Count) admin(s)"
                } else {
                    Write-Warning "    Failed to retrieve site admins: $adminJson"
                }

                $groupId = $null
                if ($site.GroupId -and $site.GroupId -notlike "*/Guid(00000000-0000-0000-0000-000000000000)/*") {
                    $groupId = $site.GroupId -replace '.*/Guid\(([^)]+)\)/.*', '$1'
                    Write-Verbose "  Site has M365 Group: $groupId"

                    Write-Verbose "  Retrieving M365 group owners..."
                    $ownersJson = m365 entra m365group user list --groupId $groupId --role Owner --output json 2>&1
                    if ($LASTEXITCODE -eq 0) {
                        $groupOwners = $ownersJson | ConvertFrom-Json
                        $reportItem.'Group Owners' = ($groupOwners.displayName | Where-Object { $_ }) -join ';'
                        Write-Verbose "    Found $($groupOwners.Count) owner(s)"
                    } else {
                        Write-Warning "    Failed to retrieve group owners: $ownersJson"
                    }

                    Write-Verbose "  Retrieving M365 group members..."
                    $membersJson = m365 entra m365group user list --groupId $groupId --role Member --output json 2>&1
                    if ($LASTEXITCODE -eq 0) {
                        $groupMembers = $membersJson | ConvertFrom-Json
                        $reportItem.'Group Members' = ($groupMembers.displayName | Where-Object { $_ }) -join ';'
                        Write-Verbose "    Found $($groupMembers.Count) member(s)"
                    } else {
                        Write-Warning "    Failed to retrieve group members: $membersJson"
                    }

                    Write-Verbose "  Retrieving M365 group guests..."
                    $guestsJson = m365 entra m365group user list --groupId $groupId --filter "userType eq 'Guest'" --output json 2>&1
                    if ($LASTEXITCODE -eq 0) {
                        $groupGuests = $guestsJson | ConvertFrom-Json
                        $reportItem.'Group Guests' = ($groupGuests.displayName | Where-Object { $_ }) -join ';'
                        Write-Verbose "    Found $($groupGuests.Count) guest(s)"
                    } else {
                        Write-Warning "    Failed to retrieve group guests: $guestsJson"
                    }
                } else {
                   Write-Verbose "  Site has no M365 Group"
               }

                Write-Verbose "  Retrieving associated SharePoint group IDs..."
                $webJson = m365 spo web get --url $site.Url --withGroups --output json 2>&1
               if ($LASTEXITCODE -eq 0) {
                    $web = $webJson | ConvertFrom-Json
                    $associatedOwnerGroupId = $web.AssociatedOwnerGroup.Id
                    $associatedMemberGroupId = $web.AssociatedMemberGroup.Id
                    $associatedVisitorGroupId = $web.AssociatedVisitorGroup.Id
                    Write-Verbose "    Associated group IDs - Owner: $associatedOwnerGroupId, Member: $associatedMemberGroupId, Visitor: $associatedVisitorGroupId"

                    # Process Owner group
                    if ($associatedOwnerGroupId) {
                        Write-Verbose "    Processing Owner group..."
                        $membersJson = m365 spo group member list --webUrl $site.Url --groupId $associatedOwnerGroupId --output json 2>&1
                       if ($LASTEXITCODE -eq 0) {
                            $ownerMembers = $membersJson | ConvertFrom-Json
                            $reportItem.'Site owners' = ($ownerMembers.Title | Where-Object { $_ }) -join ';'
                            Write-Verbose "      Found $($ownerMembers.Count) owner(s)"
                        } else {
                            Write-Warning "      Failed to retrieve Owner group members: $membersJson"
                        }
                    }

                    # Process Member group
                    if ($associatedMemberGroupId) {
                        Write-Verbose "    Processing Member group..."
                        $membersJson = m365 spo group member list --webUrl $site.Url --groupId $associatedMemberGroupId --output json 2>&1
                        if ($LASTEXITCODE -eq 0) {
                            $memberMembers = $membersJson | ConvertFrom-Json
                            $reportItem.'Site members' = ($memberMembers.Title | Where-Object { $_ }) -join ';'
                            Write-Verbose "      Found $($memberMembers.Count) member(s)"
                        } else {
                            Write-Warning "      Failed to retrieve Member group members: $membersJson"
                        }
                    }

                    # Process Visitor group
                    if ($associatedVisitorGroupId) {
                        Write-Verbose "    Processing Visitor group..."
                        $membersJson = m365 spo group member list --webUrl $site.Url --groupId $associatedVisitorGroupId --output json 2>&1
                        if ($LASTEXITCODE -eq 0) {
                            $visitorMembers = $membersJson | ConvertFrom-Json
                            $reportItem.'Site visitors' = ($visitorMembers.Title | Where-Object { $_ }) -join ';'
                            Write-Verbose "      Found $($visitorMembers.Count) visitor(s)"
                       } else {
                            Write-Warning "      Failed to retrieve Visitor group members: $membersJson"
                       }
                   }
               } else {
                    Write-Warning "    Failed to retrieve web properties: $webJson"
                }

                $script:ReportCollection.Add($reportItem)
                $script:Summary.Successes++
                Write-Host "  ✓ Completed successfully" -ForegroundColor Green
            }
            catch {
                $script:Summary.Failures++
                Write-Warning "  Failed to process site '$($site.Title)': $_"
                continue
            }

            Write-Host ""
        }
    }
    catch {
        Write-Error "An error occurred during processing: $_"
        throw
    }
}

end {
    Write-Host "`n" -NoNewline
    Write-Host "=" * 60 -ForegroundColor Cyan
    Write-Host "SITES MEMBERSHIP REPORT SUMMARY" -ForegroundColor Cyan
    Write-Host "=" * 60 -ForegroundColor Cyan
    Write-Host "  Sites scanned:         $($script:Summary.SitesScanned)" -ForegroundColor White
    Write-Host "  Successfully processed: " -NoNewline -ForegroundColor White
    Write-Host $script:Summary.Successes -ForegroundColor Green
    Write-Host "  Failed:                " -NoNewline -ForegroundColor White
    if ($script:Summary.Failures -gt 0) {
        Write-Host $script:Summary.Failures -ForegroundColor Red
    } else {
        Write-Host $script:Summary.Failures -ForegroundColor Green
    }
    Write-Host "=" * 60 -ForegroundColor Cyan

    if ($script:ReportCollection.Count -gt 0) {
        Write-Host "`nExporting membership report to CSV..." -ForegroundColor Yellow
        $script:ReportCollection | Sort-Object 'Site Name' | Export-Csv -Path $OutputPath -NoTypeInformation -Force
        Write-Host "✓ Report exported successfully" -ForegroundColor Green
        Write-Host "  Report location: $OutputPath" -ForegroundColor Cyan
    } else {
        Write-Host "`nNo membership data to export" -ForegroundColor Yellow
    }

    Write-Host "`nLog file location: $logFile" -ForegroundColor Cyan

    Stop-Transcript | Out-Null
}
```

# Usage Examples
# Example 1: Generate report for all /sites/ sites with verbose output
# .\Get-SitesMembershipReport.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -Verbose

# Example 2: Custom filter for specific sites (Dev, Test, UAT) and specify output path
# .\Get-SitesMembershipReport.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -SiteFilter "(Url -like '/sites/Dev-' or Url -like '/sites/Test-' or Url -like '/sites/Uat-') and Template ne 'RedirectSite#0'" -OutputPath "C:\Reports\CustomSites.csv"

# Example 3: Report for all Team sites (GROUP#0 template)
# .\Get-SitesMembershipReport.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -SiteFilter "Template eq 'GROUP#0'"

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell
$AdminCenterURL="https://contoso-admin.sharepoint.com/"# Connect to SharePoint Online admin center
Connect-PnPOnline -Url $AdminCenterURL -Interactive
$dateTime = (Get-Date).toString("dd-MM-yyyy")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "m365GroupUsersReport-" + $dateTime + ".csv"
$OutPutView = $directorypath + "\Logs\"+ $fileName
# Array to Hold Result - PSObjects
$m365GroupCollection = @()
#Amend query to retrieve the sites within tenant
$m365Sites = Get-PnPTenantSite -Detailed | Where-Object {($_.Url -like '*/Dev-*' -or  $_.Url -like '*/Test-*' -or  $_.Url -like '*/Uat-*' -or $_.Template -eq 'TEAMCHANNEL#1') -and $_.Template -ne 'RedirectSite#0' }

$m365Sites | ForEach-Object {
    $ExportVw = New-Object PSObject
    $ExportVw | Add-Member -MemberType NoteProperty -name "Site Name" -value $_.Title
    $m365GroupOwnersName="";
    $m365GroupMembersName="";
    $m365GroupGuestsName = "";
    $groupId = $_.GroupId;
    $siteUrl = $_.Url;
    #Check if site template is a Team template to retrieve the m365 group membership
    if($_.Template -eq "GROUP#0")
    {
        $m365GroupOwnersName = (Get-PnPMicrosoft365GroupOwner -Identity $groupId -ErrorAction Ignore| select-object -ExpandProperty DisplayName ) -join ";";
        $m365GroupMembersName = (Get-PnPMicrosoft365GroupMember -Identity $groupId  -ErrorAction Ignore| select-object -ExpandProperty DisplayName) -join ";";
        $m365GroupGuestsName = (Get-PnPMicrosoft365GroupMember -Identity $groupId  -ErrorAction Ignore |Where-Object UserType -eq Guest | select-object -ExpandProperty DisplayName) -join ";";
    }

    $ExportVw | Add-Member -MemberType NoteProperty -name "Group Owners" -value $m365GroupOwnersName    
    $ExportVw | Add-Member -MemberType NoteProperty -name "Group Members" -value $m365GroupMembersName
    $ExportVw | Add-Member -MemberType NoteProperty -name "Group Guests" -value $m365GroupGuestsName      
    Connect-PnPOnline -Url $siteUrl -Interactive
    
    $site = Get-PnPSite -Includes ID
    $ExportVw | Add-Member -MemberType NoteProperty -name "Site Id" -value $site.Id  
    $siteadmins = (Get-PnPSiteCollectionAdmin | select-object -ExpandProperty Title) -join ";";
    $ExportVw | Add-Member -MemberType NoteProperty -name "Site admins" -value $siteadmins  
    $siteowners  = (Get-PnPGroupMember -Group (Get-PnPGroup -AssociatedOwnerGroup)  | select-object -ExpandProperty Title) -join ";"
    $ExportVw | Add-Member -MemberType NoteProperty -name "Site owners" -value $siteowners
    $sitemembers =  (Get-PnPGroupMember -Group (Get-PnPGroup -AssociatedMemberGroup)  | select-object -ExpandProperty Title) -join ";"
    $ExportVw | Add-Member -MemberType NoteProperty -name "Site members" -value  $sitemembers
    $sitevisitors =  (Get-PnPGroupMember -Group (Get-PnPGroup -AssociatedVisitorGroup)  | select-object -ExpandProperty Title) -join ";"
    $ExportVw | Add-Member -MemberType NoteProperty -name "Site visitors" -value  $sitevisitors
    $m365GroupCollection += $ExportVw

}
# Export the result array to CSV file
$m365GroupCollection | sort-object "Site Name" |Export-CSV $OutPutView -Force -NoTypeInformation
# Disconnect SharePoint online connection
Disconnect-PnPOnline
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011)|
| [Adam Wójcik](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-sites-membership-report" aria-hidden="true" />

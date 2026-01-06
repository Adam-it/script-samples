

# Get the report of the sites throughout the tenant which has unique permissions based on the RoleAssignments and the Associated member groups


## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Copy a script  below
- Run the script from Windows PowerShell ISE

![Example Screenshot](assets/example.png)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding()]
param (
    [Parameter(Mandatory, HelpMessage = "SharePoint tenant admin center URL (e.g., https://contoso-admin.sharepoint.com)")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$TenantAdminUrl,

    [Parameter(HelpMessage = "Type of sites to analyze (TeamSite, CommunicationSite, or Both)")]
    [ValidateSet('TeamSite', 'CommunicationSite', 'Both')]
    [string]$SiteType = 'TeamSite',

    [Parameter(HelpMessage = "Directory path for CSV export (default: current directory)")]
    [string]$OutputPath
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = "GetSitesWithUniquePermissions_$timestamp.log"
    Start-Transcript -Path $transcriptPath

    if ($OutputPath) {
        if (-not (Test-Path -Path $OutputPath -PathType Container)) {
            throw "Output path '$OutputPath' does not exist or is not a directory."
        }
    } else {
        $OutputPath = (Get-Location).Path
    }

    Write-Host "[1/4] Ensuring CLI for Microsoft 365 login..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Exit code: $LASTEXITCODE"
    }
    Write-Host "Successfully authenticated." -ForegroundColor Green

    $script:ReportCollection = @()
    $script:Summary = @{
        SitesProcessed = 0
        SitesWithUniquePermissions = 0
        Failures = 0
    }
}

process {
    try {
        Write-Host "`n[2/4] Retrieving sites from tenant..." -ForegroundColor Cyan
        $siteTypeParam = switch ($SiteType) {
            'TeamSite' { 'TeamSite' }
            'CommunicationSite' { 'CommunicationSite' }
            'Both' { 'TeamSite,CommunicationSite' }
        }
        
        $sitesJson = m365 spo site list --type $siteTypeParam --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sites. Exit code: $LASTEXITCODE"
        }

        $sites = @($sitesJson | ConvertFrom-Json)
        Write-Host "Found $($sites.Count) site(s) to process ($SiteType)." -ForegroundColor Green

        if ($sites.Count -eq 0) {
            Write-Host "No sites found in tenant. Exiting." -ForegroundColor Yellow
            return
        }

        Write-Host "`n[3/4] Analyzing permissions for each site..." -ForegroundColor Cyan

        foreach ($site in $sites) {
            try {
                Write-Verbose "Processing site: $($site.Url)"
                $script:Summary.SitesProcessed++

                $webJson = m365 spo web get --url $site.Url --withPermissions --withGroups --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve web details for '$($site.Url)'. CLI: $webJson"
                    $script:Summary.Failures++
                    continue
                }

                $web = $webJson | ConvertFrom-Json

                $roleAssignmentsChanged = $false
                $membersGroupChanged = $false

                if ($web.RoleAssignments -and $web.RoleAssignments.Count -gt 3) {
                    $roleAssignmentsChanged = $true
                    Write-Verbose "Site has $($web.RoleAssignments.Count) role assignments (> 3)."
                }

                if ($web.AssociatedMemberGroup -and $web.AssociatedMemberGroup.Id) {
                    $membersJson = m365 spo group member list --webUrl $site.Url --groupId $web.AssociatedMemberGroup.Id --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to retrieve member group users for '$($site.Url)'. CLI: $membersJson"
                        $script:Summary.Failures++
                        continue
                    }

                    $members = @($membersJson | ConvertFrom-Json)
                    if ($members.Count -gt 1) {
                        $membersGroupChanged = $true
                        Write-Verbose "Associated member group has $($members.Count) member(s) (> 1)."
                    }
                }

                if ($roleAssignmentsChanged -or $membersGroupChanged) {
                    $script:Summary.SitesWithUniquePermissions++
                    $script:ReportCollection += [PSCustomObject]@{
                        SiteUrl = $site.Url
                        IsRoleAssignmentsChanged = $roleAssignmentsChanged
                        IsMembersGroupChanged = $membersGroupChanged
                    }
                    Write-Verbose "Site has unique permissions."
                }
            } catch {
                Write-Warning "Error processing site '$($site.Url)': $_"
                $script:Summary.Failures++
                continue
            }
        }
    } catch {
        Write-Error "Critical error during execution: $_"
        throw
    }
}

end {
    Write-Host "`n[4/4] Exporting results and summary..." -ForegroundColor Cyan

    if ($script:ReportCollection.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "GetSitesWithUniquePermissions_$timestamp.csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -Encoding UTF8 -NoTypeInformation -Delimiter ";"
        Write-Host "Report exported to: " -NoNewline
        Write-Host $csvPath -ForegroundColor Cyan
    } else {
        Write-Host "No sites with unique permissions found. No CSV exported." -ForegroundColor Yellow
    }

    Write-Host "`n========== SUMMARY =========="
    Write-Host "Sites Processed:                 $($script:Summary.SitesProcessed)"
    Write-Host "Sites with Unique Permissions:   " -NoNewline
    Write-Host $script:Summary.SitesWithUniquePermissions -ForegroundColor Cyan
    Write-Host "Failures:                        " -NoNewline
    if ($script:Summary.Failures -gt 0) {
        Write-Host $script:Summary.Failures -ForegroundColor Red
    } else {
        Write-Host $script:Summary.Failures -ForegroundColor Green
    }
    Write-Host "=============================`n"

    Write-Host "Transcript log saved to: " -NoNewline
    Write-Host $transcriptPath -ForegroundColor Cyan

    Stop-Transcript
}

# Basic usage
# .\.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com"

# Analyze Communication sites only
# .\.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -SiteType CommunicationSite

# Analyze both Team and Communication sites
# .\.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -SiteType Both

# With custom output path and verbose logging
# .\.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -SiteType Both -OutputPath "C:\Reports" -Verbose

# GCC High tenant
# .\.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.us" -SiteType Both

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

<#
This script is used to get sites with unique permissions
#>

$generatedCSVPath = "GetSitesWithUniquePermissions.csv"
$tenantUrl ="<Tenant Site URL>"

function GetSitesWithUniquePermissions {
    Connect-PnPOnline -Url $tenantUrl -Interactive
    
    #Here we are targetting just the Team sites in the specified tenant
    $sites = Get-PnPTenantSite -Template GROUP#0 
    foreach ($site in $sites){
        Connect-PnPOnline -Url $site.Url -Interactive
        $web = Get-PnPWeb -Includes RoleAssignments
        <#
        Check if the RoleAssignments count is greater than 3. 
        If true, then this site does has more than default roleassignments. 
        Returns boolean
        #>
        $moreThanDefaultRoleAssignments = ($web.RoleAssignments.Count -gt 3)

        $group = Get-PnPGroup -AssociatedMemberGroup
        <#
        Checks if the users in the associate member group is greater than 1 user. 
        Returns boolean
        #>
        $usersCount = ($group.Users.Count -gt 1)

        [PSCustomObject]@{
            "SiteUrl"        = $site.Url
            "IsRoleAssigmentsChanged" = $moreThanDefaultRoleAssignments
            "IsMembersGroupChanged" = $usersCount
        } | Export-Csv -Path $generatedCSVPath -Encoding UTF8 -NoTypeInformation -Delimiter ";" -Append
    }
}


GetSitesWithUniquePermissions


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

![Preview Screenshot](assets/preview.png)


## Contributors

| Author(s) |
|-----------|
| [Nishkalank Bezawada](https://github.com/NishkalankBezawada) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-sites-with-unique-permissions" aria-hidden="true" />

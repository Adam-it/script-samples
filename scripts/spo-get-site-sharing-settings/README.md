

# Get SharePoint site sharing Settings

## Summary

Reviewing sharing settings is essential to prevent oversharing, which can lead to data breaches and unauthorised access to sensitive information. By regularly auditing and adjusting these settings, organization's sharing policies can be enforced and ensure that only authorized users have access to specific content. This is particularly important during the rollout of Copilot for M365, as it helps maintain a secure and compliant environment.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have SharePoint Online site administrator and SharePoint Administrator access .

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Get-SpoSiteSharingSettings {
    [CmdletBinding(SupportsShouldProcess)]
    param (
        [Parameter(Mandatory = $true, HelpMessage = "SharePoint tenant domain, e.g. 'contoso' for contoso.sharepoint.com.")]
        [ValidatePattern('^[a-z0-9-]+$')]
        [string]$TenantDomain,

        [Parameter(Mandatory = $false, HelpMessage = "Write report to this CSV file (defaults to current directory with timestamp).")]
        [string]$OutputPath
    )

    begin {
        Write-Verbose 'Ensuring CLI for Microsoft 365 authentication'
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to sign in with CLI for Microsoft 365. CLI output: $loginOutput"
        }

        $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
        if (-not $OutputPath) {
            $OutputPath = Join-Path -Path (Get-Location) -ChildPath "site_sharing_settings_$timestamp.csv"
        }

        $directory = Split-Path -Path $OutputPath -Parent
        if ($directory -and -not (Test-Path -Path $directory -PathType Container)) {
            Write-Verbose "Creating export directory '$directory'"
            New-Item -ItemType Directory -Path $directory -Force | Out-Null
        }

        $script:Summary = [ordered]@{
            SitesProcessed = 0
            Failures       = @()
            ReportPath     = $OutputPath
            Records        = @()
        }

        $tenantHost = "https://$TenantDomain.sharepoint.com"
        $script:SiteQuery = "[?starts_with(url, '$tenantHost') && template!='RedirectSite#0']"
    }

    process {
        Write-Verbose 'Retrieving SharePoint sites'
        $siteOutput = m365 spo site list --output json --query $script:SiteQuery 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list sites. CLI output: $siteOutput"
        }

        $sites = @()
        if (-not [string]::IsNullOrWhiteSpace($siteOutput)) {
            $sites = @($siteOutput | ConvertFrom-Json)
        }

        if (-not $sites) {
            throw "No SharePoint sites found under $tenantHost"
        }

        foreach ($site in $sites) {
            $siteUrl = $site.url
            Write-Verbose "Processing $siteUrl"

            $siteDetailsOutput = m365 spo site get --url $siteUrl --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Failures += [pscustomobject]@{ Stage='SiteDetails'; Target=$siteUrl; Message=$siteDetailsOutput }
                Write-Warning "Failed to retrieve site details for $siteUrl. CLI output: $siteDetailsOutput"
                continue
            }

            $siteDetails = @()
            if (-not [string]::IsNullOrWhiteSpace($siteDetailsOutput)) {
                $siteDetails = @($siteDetailsOutput | ConvertFrom-Json)
            }
            if (-not $siteDetails) { continue }
            $siteInfo = $siteDetails[0]

            $webOutput = m365 spo web get --webUrl $siteUrl --output json --select MembersCanShare TenantAdminMembersCanShare RequestAccessEmail UseAccessRequestDefault AccessRequestSiteDescription 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Failures += [pscustomobject]@{ Stage='WebProperties'; Target=$siteUrl; Message=$webOutput }
                Write-Warning "Failed to retrieve additional web properties for $siteUrl. CLI output: $webOutput"
                continue
            }

            $webProps = @{}
            if (-not [string]::IsNullOrWhiteSpace($webOutput)) {
                $webProps = $webOutput | ConvertFrom-Json
            }

            $record = [pscustomobject]@{
                Title                                 = $siteInfo.title
                Url                                   = $siteInfo.url
                Template                              = $siteInfo.template
                SharingCapability                     = $siteInfo.sharingCapability
                SharingDomainRestrictionMode          = $siteInfo.sharingDomainRestrictionMode
                SharingAllowedDomainList              = $siteInfo.sharingAllowedDomainList
                SharingBlockedDomainList              = $siteInfo.sharingBlockedDomainList
                DefaultSharingLinkType                = $siteInfo.defaultSharingLinkType
                DefaultLinkPermission                 = $siteInfo.defaultLinkPermission
                DefaultShareLinkScope                 = $siteInfo.defaultShareLinkScope
                DefaultShareLinkRole                  = $siteInfo.defaultShareLinkRole
                DisableCompanyWideSharingLinks        = $siteInfo.disableCompanyWideSharingLinks
                DisableSharingForNonOwnersStatus      = $siteInfo.disableSharingForNonOwnersStatus
                ExternalUserExpirationInDays          = $siteInfo.externalUserExpirationInDays
                AnonymousLinkExpirationInDays         = $siteInfo.anonymousLinkExpirationInDays
                OverrideTenantExternalUserExpiration  = $siteInfo.overrideTenantExternalUserExpirationPolicy
                OverrideTenantAnonymousLinkExpiration = $siteInfo.overrideTenantAnonymousLinkExpirationPolicy
                ConditionalAccessPolicy               = $siteInfo.conditionalAccessPolicy
                ReadOnlyForUnmanagedDevices           = $siteInfo.readOnlyForUnmanagedDevices
                RequestFilesLinkEnabled               = $siteInfo.requestFilesLinkEnabled
                RequestFilesLinkExpirationInDays      = $siteInfo.requestFilesLinkExpirationInDays
                RestrictContentOrgWideSearch          = $siteInfo.restrictContentOrgWideSearch
                MembersCanShare                       = $webProps.MembersCanShare
                TenantAdminMembersCanShare            = $webProps.TenantAdminMembersCanShare
                RequestAccessEmail                    = $webProps.RequestAccessEmail
                UseAccessRequestDefault               = $webProps.UseAccessRequestDefault
                AccessRequestSiteDescription          = $webProps.AccessRequestSiteDescription
            }

            $script:Summary.Records += $record
            $script:Summary.SitesProcessed++
        }
    }

    end {
        if (-not $script:Summary.Records) {
            Write-Warning 'No site sharing settings were collected.'
            return [pscustomobject]$script:Summary
        }

        if ($PSCmdlet.ShouldProcess($script:Summary.ReportPath, 'Export site sharing report')) {
            Write-Verbose "Exporting site sharing report to '$($script:Summary.ReportPath)'"
            $script:Summary.Records | Export-Csv -Path $script:Summary.ReportPath -NoTypeInformation
        }

        Write-Host '--- Site sharing summary ---'
        Write-Host "Sites processed : $($script:Summary.SitesProcessed)"
        Write-Host "Report file     : $($script:Summary.ReportPath)"
        Write-Host "Failures        : $($script:Summary.Failures.Count)"

        return [pscustomobject]$script:Summary
    }
}

Get-SpoSiteSharingSettings -TenantDomain 'contoso' -Verbose

```

# [PnP PowerShell](#tab/pnpps)

```powershell
param (
    [Parameter(Mandatory = $true)]
    [string] $domain
)

$adminSiteURL = "https://$domain-Admin.SharePoint.com"
$TenantURL = "https://$domain.sharepoint.com"
$dateTime = "_{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "site_sharing_settings" + $dateTime + ".csv"
$outputPath = $directorypath + "\"+ $fileName

if (-not (Test-Path $outputPath)) {
    New-Item -ItemType File -Path $outputPath
}
Connect-PnPOnline -Url $adminSiteURL -Interactive -WarningAction SilentlyContinue
$adminConnection = Get-PnPConnection
        Write-Host "Getting site sharing settings..." -ForegroundColor Yellow
        $sharingReport = Get-PnPTenantSite -Filter "Url -like '$TenantURL'" | Where-Object { $_.Template -ne 'RedirectSite#0' }  | foreach-object {
          try {    
            $sharingsetting = Get-PnPTenantSite -url $_.Url -DisableSharingForNonOwnersStatus -Connection $adminConnection| select `
            Title, `
            Url, `
            Type, `
            Template, `
            ShowPeoplePickerSuggestionsForGuestUsers, `
            SharingCapability, `
            ExternalUserExpirationInDays, `
            SharingAllowedDomainList, `
            SharingBlockedDomainList, `
            SharingDomainRestrictionMode, `
            OverrideTenantExternalUserExpirationPolicy, `
            OverrideTenantAnonymousLinkExpirationPolicy, `
            DefaultSharingLinkType, `
            DefaultLinkPermission, `
            DefaultShareLinkScope, `
            DefaultShareLinkRole, `
            DefaultLinkToExistingAccess, `
            DisableCompanyWideSharingLinks, `
            DisableSharingForNonOwnersStatus, `
            AnonymousLinkExpirationInDays, `
            ConditionalAccessPolicy, `
            ReadOnlyForUnmanagedDevices, `
            LoopDefaultSharingLinkScope, `
            LoopDefaultSharingLinkRole, `
            OverrideSharingCapability, `
            RequestFilesLinkEnabled, `
            RequestFilesLinkExpirationInDays, `
            RestrictedAccessControl, `
            RestrictedAccessControlGroups, `
            RestrictContentOrgWideSearch, `
            SensitivityLabel
            # DefaultShareLinkScope and DefaultShareLinkRole will replace DefaultSharingLinkType and DefaultLinkPermission
            $restUrl = $_.Url +'/_api/web?$select=MembersCanShare,TenantAdminMembersCanShare,RequestAccessEmail,UseAccessRequestDefault,AccessRequestSiteDescription'
            connect-PnPOnline -Url $_.Url -interactive -WarningAction SilentlyContinue
            $siteconnection = Get-PnPConnection
            $response = invoke-pnpsprestmethod -Url $restUrl -Method Get -Connection $siteconnection
            $groupType = ""
            $allowToAddGuests = $null;
            $m365Group = $null;
            #find if the site is linked to a m365 group and retrieve visibility 
            if($_.groupId  -ne [guid]::Empty){
                $m365Group = Get-PnPMicrosoft365Group -Identity $_.groupId -Connection $adminConnection | select Visibility
                $m365GroupSettings =  Get-PnPMicrosoft365GroupSettings -Identity $_.GroupId -Connection $adminConnection
                $allowToAddGuests = $m365GroupSettings.Values | Where-Object {$_.Name -eq 'AllowToAddGuests'}
                #Get group type (group, team, yammer)
                $gEndPoint = Get-PnPMicrosoft365GroupEndpoint -Identity $_.groupId
                $groupType = $gEndPoint ? $gEndPoint.Providername : "SharePoint Team Site or Outlook";
                #Get guest user count
                #$settings = New-PnPMicrosoft365GroupSettings  -Identity $_.groupId -DisplayName "Group.Unified.Guest" -TemplateId "08d542b9-071f-4e16-94b0-74abb372e3d9" -Values @{"AllowToAddGuests"="false"} 
            }

 
            [PSCustomObject]@{
                ##add the properties from the $sharingsetting object
                Title = $sharingsetting.Title
                Url = $sharingsetting.Url
                ShowPeoplePickerSuggestionsForGuestUsers = $sharingsetting.ShowPeoplePickerSuggestionsForGuestUsers
                SharingCapability = $sharingsetting.SharingCapability
                ExternalUserExpirationInDays = $sharingsetting.ExternalUserExpirationInDaysre
                SharingAllowedDomainList = $sharingsetting.SharingAllowedDomainList
                SharingBlockedDomainList = $sharingsetting.SharingBlockedDomainList
                SharingDomainRestrictionMode = $sharingsetting.SharingDomainRestrictionMode
                OverrideTenantExternalUserExpirationPolicy = $sharingsetting.OverrideTenantExternalUserExpirationPolicy
                DefaultSharingLinkType = $sharingsetting.DefaultSharingLinkType
                DefaultLinkPermission = $sharingsetting.DefaultLinkPermission
                DefaultShareLinkScope  = $sharingsetting.DefaultShareLinkScope
                DefaultShareLinkRole = $sharingsetting.DefaultShareLinkRole
                DefaultLinkToExistingAccess = $sharingsetting.DefaultLinkToExistingAccess
                DisableCompanyWideSharingLinks = $sharingsetting.DisableCompanyWideSharingLinks
                AnonymousLinkExpirationInDays = $sharingsetting.AnonymousLinkExpirationInDays
                ConditionalAccessPolicy = $sharingsetting.ConditionalAccessPolicy
                ReadOnlyForUnmanagedDevices = $sharingsetting.ReadOnlyForUnmanagedDevices
                LoopDefaultSharingLinkScope = $sharingsetting.LoopDefaultSharingLinkScope
                LoopDefaultSharingLinkRole = $sharingsetting.LoopDefaultSharingLinkRole
                OverrideSharingCapability = $sharingsetting.OverrideSharingCapability
                OverrideTenantAnonymousLinkExpirationPolicy = $sharingsetting.OverrideTenantAnonymousLinkExpirationPolicy
                RequestFilesLinkEnabled = $sharingsetting.RequestFilesLinkEnabled
                RequestFilesLinkExpirationInDays = $sharingsetting.RequestFilesLinkExpirationInDays
                RestrictContentOrgWideSearch = $sharingsetting.RestrictContentOrgWideSearch
                DisableSharingForNonOwners = $sharingsetting.DisableSharingForNonOwnersStatus
                SensitivityLabel = $sharingsetting.SensitivityLabel
                SiteType = If($sharingsetting.Template -eq "GROUP#0"){"Group"} elseif ($sharingsetting.Template -eq "TEAMCHANNEL#1" -or $sharingsetting.Template -eq "TEAMCHANNEL#0"){"Team Channel"} else {"Site"}
                ##add the properties from the $response object
                MembersCanShare = $response.MembersCanShare
                TenantAdminMembersCanShare = $response.TenantAdminMembersCanShare
                RequestAccessEmail = $response.RequestAccessEmail
                UseAccessRequestDefault = $response.UseAccessRequestDefault
                AccessRequestSiteDescription = $response.AccessRequestSiteDescription
                ##add m365 group settings if site is linked to a m365 group
                m365GroupId = if($_.groupId -ne [guid]::Empty){$_.groupId}
                m365GroupVisibility = $m365Group.Visibility
                m365GroupAllowToAddGuests = $allowToAddGuests.Value ?? "Default"
                m365GroupType = $groupType
            }
        }
        catch {
            Write-Host "An error occurred: $_" -ForegroundColor Red
        }     
    }
    $sharingReport |select *  |Export-Csv $outputPath -NoTypeInformation -Append
    Write-Host "Exported successfully!..." -ForegroundColor Green
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Get SharePoint site sharing Settings with PowerShell](https://reshmeeauckloo.com/posts/powershell-sharing-settings-sharepoint-site/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-site-sharing-settings" aria-hidden="true" />

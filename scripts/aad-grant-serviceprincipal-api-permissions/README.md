

# Grant Managed Identity permissions to audit and cleanup "SharePoint Online Client Extensibility Web Application Principal" API permissions

## Summary

 This script can be used to grant System-Managed Identity used by automation (Azure Runbook, Azure Functions) API permissions and access to SPO sites,that are necessary to:
- audit API permissions assigned to the "SharePoint Online Client Extensibility Web Application Principal".
- audit permissions requested by the installed SPFx solutions (tenant-level and site-level app catalogs)
- remove unused permissions.

### Set-ManagedIdentityAPIPermissions
The `Set-ManagedIdentityAPIPermissions` function grants the following roles to the Managed Identity used for automation:
    - `Application.Read.All`,
    - `Sites.Selected`     ,
    - `DelegatedPermissionGrant.ReadWrite.All`

Once the `Lists.SelectedOperations.Selected` is available productively, the `Sites.Selected` scope can be replaced.

![API permissions](./assets/image.png)

### Set-SiteAppCatalogPermissions
The `Set-SiteAppCatalogPermissions` function grants the Service Principal read access to 
- root level SharePoint Site
- tenant-level app catalog
- sites with site-level app catalog

> The script uses **Microsoft Graph PowerShell** (`Set-ManagedIdentityAPIPermissions`) and **PnP PowerShell** (`Set-SiteAppCatalogPermissions`)
>
> PnP PowerShell requires PowerShell 7.2 or later.

## Resources
Overview of Selected permissions in OneDrive and SharePoint: https://learn.microsoft.com/en-us/graph/permissions-selected-overview?tabs=powershell
Assigning application permissions to lists, list items, folders, or files breaks inheritance on the assigned resource, 
so be mindful of service limits for unique permissions in your solution design. Permissions at the site collection level do not break inheritance 
because this is the root of permission inheritance.

Other useful tools:
- Microsoft Graph Permissions Explorer: https://graphpermissions.merill.net/permission/
- Export-MsIdAppConsentGrantReport https://azuread.github.io/MSIdentityTools/commands/Export-MsIdAppConsentGrantReport

## Important

To assign all permissions, please execute both functions:

```powershell
param(
    [string]$spId,
    [string]$tenantName
)
Set-ManagedIdentityAPIPermissions -spId $spId 
Set-SiteAppCatalogPermissions -tenantName $tenantName -spId $spId 
```
The below tabs have different contents:
- PnP PowerShell: `Set-SiteAppCatalogPermissions` function
. Microsoft Graph PowerShell: `Set-ManagedIdentityAPIPermissions` function

# [PnP PowerShell](#tab/pnpps)

```powershell
<#
    .DESCRIPTION
    The Set-SiteAppCatalogPermissions function grants Managed Identity Read acess to the following SPO sites:
    - root site : this is required for the Azure Runbook to connect to SharePoint and request app catalogs
    - tenant level app catalog
    - all detected site level app catalogs
#>
function Set-SiteAppCatalogPermissions {
    param(
        [string]$tenantName,
        [string]$spId
    )
    $adminUrl = "https://$tenantName-admin.sharepoint.com/"

    Import-Module PnP.PowerShell
    Write-Host "Connect to SharePoint Admin site: $adminUrl "
    Connect-PnPOnline -Url $adminUrl -Interactive

    # get Service Principal to retrieve AppId
    $sp = Get-MgServicePrincipal -ServicePrincipalId $spId #script will stop if service principal does not exist

    Get-PnPSiteCollectionAppCatalog -ExcludeDeletedSites -PipelineVariable SiteAppCatalog | ForEach-Object {
        Grant-PnPAzureADAppSitePermission -AppId $sp.AppId -DisplayName $sp.DisplayName -Permissions Read -Site $SiteAppCatalog.SiteID.Guid
    }
    $tenantLevelAppCatalog = Get-PnPTenantAppCatalogUrl
    Grant-PnPAzureADAppSitePermission -AppId $sp.AppId -DisplayName $sp.DisplayName -Permissions Read -Site $tenantLevelAppCatalog
    Grant-PnPAzureADAppSitePermission -AppId $sp.AppId -DisplayName $sp.DisplayName -Permissions Read -Site "https://$tenantName.sharepoint.com/"

}
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]




# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
function Grant-AadServicePrincipalApiPermission {
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter(Mandatory, HelpMessage = "Object ID of the service principal that will receive the permissions.")]
        [ValidateNotNullOrEmpty()]
        [string]$ServicePrincipalId,

        [Parameter(Mandatory, HelpMessage = "Display name of the resource API (for example 'Microsoft Graph').")]
        [ValidateNotNullOrEmpty()]
        [string]$ResourceDisplayName,

        [Parameter(Mandatory, HelpMessage = "Application (app-only) scopes to grant.")]
        [ValidateNotNullOrEmpty()]
        [string[]]$ApplicationScopes,

        [Parameter(HelpMessage = "Delegated scopes to grant (optional).")]
        [string[]]$DelegatedScopes,

        [Parameter(HelpMessage = "Skip confirmation prompts and grant permissions immediately.")]
        [switch]$Force
    )

    begin {
        $script:Summary = [ordered]@{
            ApplicationGranted = 0
            DelegatedGranted   = 0
            Skipped            = 0
            Failures           = 0
        }

        if (-not $PSBoundParameters.ContainsKey('DelegatedScopes')) {
            $DelegatedScopes = @()
        }

        Write-Host "Ensuring Microsoft 365 CLI authentication." -ForegroundColor Cyan
        $loginResult = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Authentication with CLI for Microsoft 365 failed. CLI output: $loginResult"
        }

        Write-Host "Resolving resource service principal '$ResourceDisplayName'." -ForegroundColor Cyan
        $resourceJson = m365 entra serviceprincipal list --displayName $ResourceDisplayName --query "[0]" --output json 2>&1
        if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace(($resourceJson -as [string]).Trim())) {
            throw "Resource '$ResourceDisplayName' was not found. CLI output: $resourceJson"
        }

        try {
            $script:ResourceServicePrincipal = $resourceJson | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse resource service principal. $($_.Exception.Message)"
        }
        Write-Host "Using resource appId $($script:ResourceServicePrincipal.appId)." -ForegroundColor DarkCyan

        $grantsResult = m365 entra serviceprincipal apppermission list --id $ServicePrincipalId --query "[?resourceAppId=='$($script:ResourceServicePrincipal.appId)']" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Unable to retrieve existing application grants. CLI output: $grantsResult"
            $script:ExistingAppGrants = @()
        }
        else {
            $script:ExistingAppGrants = $grantsResult | ConvertFrom-Json
        }

        $delegatedResult = m365 entra serviceprincipal oauth2permissiongrant list --id $ServicePrincipalId --query "[?resourceId=='$($script:ResourceServicePrincipal.id)']" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Unable to retrieve existing delegated grants. CLI output: $delegatedResult"
            $script:ExistingDelegatedGrants = @()
        }
        else {
            $script:ExistingDelegatedGrants = $delegatedResult | ConvertFrom-Json
        }

    }

    process {
        foreach ($scope in $ApplicationScopes) {
            if ([string]::IsNullOrWhiteSpace($scope)) {
                continue
            }

            $alreadyGranted = $script:ExistingAppGrants | Where-Object {
                $_.resourceAppId -eq $script:ResourceServicePrincipal.appId -and $_.permission -eq $scope
            }

            if ($alreadyGranted) {
                Write-Host "Scope '$scope' already granted (application)." -ForegroundColor Yellow
                $script:Summary.Skipped++
                continue
            }

            $shouldGrantApp = $Force.IsPresent -or $PSCmdlet.ShouldProcess($ServicePrincipalId, "Grant application permission '$scope'")
            if (-not $shouldGrantApp) {
                $script:Summary.Skipped++
                Write-Host "Skipped granting application scope '$scope' at user request." -ForegroundColor Yellow
                continue
            }

            Write-Host "Granting application scope '$scope'." -ForegroundColor Cyan
            $grantOutput = m365 entra serviceprincipal apppermission add --id $ServicePrincipalId --resource $script:ResourceServicePrincipal.appId --scope $scope --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Failures++
                Write-Warning "Failed to grant application scope '$scope'. CLI output: $grantOutput"
            }
            else {
                $script:Summary.ApplicationGranted++
                Write-Host "Granted application scope '$scope'." -ForegroundColor Green
            }
        }

        foreach ($scope in $DelegatedScopes) {
            if ([string]::IsNullOrWhiteSpace($scope)) {
                continue
            }

            $existingDelegated = $script:ExistingDelegatedGrants | Where-Object {
                $_.resourceId -eq $script:ResourceServicePrincipal.id -and $_.scope -split ' ' -contains $scope
            }

            if ($existingDelegated) {
                Write-Host "Scope '$scope' already granted (delegated)." -ForegroundColor Yellow
                $script:Summary.Skipped++
                continue
            }

            $shouldGrantDelegated = $Force.IsPresent -or $PSCmdlet.ShouldProcess($ServicePrincipalId, "Grant delegated scope '$scope'")
            if (-not $shouldGrantDelegated) {
                $script:Summary.Skipped++
                Write-Host "Skipped granting delegated scope '$scope' at user request." -ForegroundColor Yellow
                continue
            }

            if ($shouldGrantDelegated) {
                Write-Host "Granting delegated scope '$scope'." -ForegroundColor Cyan
                $delegatedOutput = m365 entra serviceprincipal oauth2permissiongrant add --id $ServicePrincipalId --resource $script:ResourceServicePrincipal.id --scope $scope --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    $script:Summary.Failures++
                    Write-Warning "Failed to grant delegated scope '$scope'. CLI output: $delegatedOutput"
                }
                else {
                    $script:Summary.DelegatedGranted++
                    Write-Host "Granted delegated scope '$scope'." -ForegroundColor Green
                }
            }
        }
    }

    end {
        Write-Host "Application scopes granted: $($script:Summary.ApplicationGranted)" -ForegroundColor Green
        Write-Host "Delegated scopes granted: $($script:Summary.DelegatedGranted)" -ForegroundColor Green
        Write-Host "Scopes skipped: $($script:Summary.Skipped)" -ForegroundColor Yellow
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red

        if (-not $DelegatedScopes -and -not $Force) {
            Write-Host "Remember to grant admin consent if required." -ForegroundColor DarkYellow
        }
    }
}

# example usage
Grant-AadServicePrincipalApiPermission -ServicePrincipalId '00000000-0000-0000-0000-000000000000' `
    -ResourceDisplayName 'Microsoft Graph' `
    -ApplicationScopes @('Application.Read.All','Sites.Selected') `
    -DelegatedScopes @('Sites.Selected') -WhatIf
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]


***



## Contributors

| Author(s) |
|-----------|
| Kinga Kazala |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/aad-grant-serviceprincipal-api-permissions" aria-hidden="true" />

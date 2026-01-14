

# Grant permissions for a given Azure Active Directory application registration

## Summary

This script simplifies the process of granting `Read`, `Write`, `Manage`, or `FullControl` permissions for an application registration in a SharePoint site collection, specifically when used in conjunction with the Azure Active Directory SharePoint application permission `Sites.Selected`.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site collection")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory, HelpMessage = "Client ID (GUID) of the Azure AD application")]
    [ValidateNotNullOrEmpty()]
    [string]$AppId,
    
    [Parameter(Mandatory, HelpMessage = "Permission level to grant (Read, Write, Manage, or FullControl)")]
    [ValidateSet('Read', 'Write', 'Manage', 'FullControl')]
    [string]$Permission
)

begin {
    Write-Host "Granting $Permission permission to app $AppId on site $SiteUrl..." -ForegroundColor Cyan
    
    # Login
    Write-Verbose "Ensuring CLI for Microsoft 365 session..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    
    # Convert permission to lowercase for CLI
    $permissionLower = $Permission.ToLower()
    
    $script:Summary = @{
        SiteUrl = $SiteUrl
        AppId = $AppId
        RequestedPermission = $Permission
        GrantedPermission = $null
        PermissionId = $null
        Success = $false
    }
    
    Start-Transcript -Path "GrantAppSitePermission_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
}

process {
   try {
        Write-Verbose "Granting $permissionLower permission..."
        $result = m365 spo site apppermission add --siteUrl $SiteUrl --permission $permissionLower --appId $AppId --output json 2>&1
        
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to grant $permissionLower permission: $result"
        }
        
        $permission = $result | ConvertFrom-Json
        
        $script:Summary.GrantedPermission = $permission.roles -join ', '
        $script:Summary.PermissionId = $permission.id
        $script:Summary.Success = $true
        
    } catch {
        Write-Error "Failed to grant permission: $_"
        throw
    }
}

end {
    Stop-Transcript
    
    # Display summary
    Write-Host "`n===== SUMMARY =====" -ForegroundColor Cyan
    Write-Host "Site URL: $($script:Summary.SiteUrl)" -ForegroundColor White
    Write-Host "App ID: $($script:Summary.AppId)" -ForegroundColor White
    Write-Host "Requested Permission: $($script:Summary.RequestedPermission)" -ForegroundColor White
    Write-Host "Granted Permission: $($script:Summary.GrantedPermission)" -ForegroundColor White
    Write-Host "Permission ID: $($script:Summary.PermissionId)" -ForegroundColor White
    Write-Host "Status: $(if ($script:Summary.Success) { 'Success ✅' } else { 'Failed ❌' })" -ForegroundColor $(if ($script:Summary.Success) { 'Green' } else { 'Red' })
    Write-Host "==================" -ForegroundColor Cyan
}

# Basic usage - grant Read permission
# .\Grant-AppSitePermission.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -AppId "5c89fbbf-670e-48e7-a0bc-fa8942c895a2" -Permission Read

# Grant FullControl permission 
# .\Grant-AppSitePermission.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -AppId "5c89fbbf-670e-48e7-a0bc-fa8942c895a2" -Permission FullControl

# Grant Manage permission
# .\Grant-AppSitePermission.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -AppId "5c89fbbf-670e-48e7-a0bc-fa8942c895a2" -Permission Manage

# With verbose output
# .\Grant-AppSitePermission.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -AppId "5c89fbbf-670e-48e7-a0bc-fa8942c895a2" -Permission Write -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
param(
    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string]$SiteUrl,

    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [string]$AppId,

    [Parameter(Mandatory)]
    [ValidateSet('Read', 'Write', 'Manage', 'FullControl')]
    [string]$Permissions
)

Connect-PnPOnline -Url $SiteUrl -Interactive

$DisplayName = (Get-PnPAzureADApp -Identity $AppId).DisplayName
if ($Permissions -eq 'FullControl' -or $Permissions -eq 'Manage') {    
    Grant-PnPAzureADAppSitePermission -Permissions Write -Site $SiteUrl -AppId $AppId -DisplayName $DisplayName | Out-Null
    $PermissionId = Get-PnPAzureADAppSitePermission -AppIdentity $AppId
    Set-PnPAzureADAppSitePermission -Site $SiteUrl -PermissionId $(($PermissionId).Id) -Permissions $Permissions | Out-Null
    Get-PnPAzureADAppSitePermission -AppIdentity $AppId
}
else {
    Grant-PnPAzureADAppSitePermission -Permissions $Permissions -Site $SiteUrl -AppId $AppId -DisplayName $DisplayName
}
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]


## Source Credit

Sample idea first appeared on [https://www.leonarmston.com/2022/02/use-sites-selected-permission-with-fullcontrol-rather-than-write-or-read/](https://www.leonarmston.com/2022/02/use-sites-selected-permission-with-fullcontrol-rather-than-write-or-read/). This is a slightly modified version.

## Contributors

| Author(s) |
|-----------|
| [Michał Romiszewski](https://github.com/mromiszewski) |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-grant-app-site-permission" aria-hidden="true" />

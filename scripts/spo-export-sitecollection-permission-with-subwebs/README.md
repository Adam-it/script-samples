

# Get SharePoint Site Collection and their Subwebs Permissions And Export It To CSV

## Summary

Sometimes we have requirement like get User Permissions Audit Report for a Site Collection with all their SubWebs. This script exports site collection and subweb permissions to CSV, supporting both PnP PowerShell and CLI for Microsoft 365 v11.4.0+.

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as below,
- First, we will Read site URL from user and connect to the Site
	- then we will get all subwebs for the site collcetion.
    - And then we will get all the permissions for site collection and subwebs.
    - Then we will export an object to CSV.
 
# [PnP PowerShell](#tab/pnpps)
```powershell

$username = "chandani@domain.onmicrosoft.com"
$password = "********"
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$Creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd
$global:permissions = @()
$BasePath = "E:\Contribution\PnP-Scripts\SitePermission\"
$DateTime = "{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$CSVPath = $BasePath + "\sitepermissions" + $DateTime + ".csv"

Function ConnectToSPSite() {
    try {
        $SiteUrl = Read-Host "Please enter Site URL"
        if ($SiteUrl) {
            Write-Host "Connecting to Site :'$($SiteUrl)'..." -ForegroundColor Yellow  
            Connect-PnPOnline -Url $SiteUrl -Credentials $Creds
            Write-Host "Connection Successfull to site: '$($SiteUrl)'" -ForegroundColor Green              
            WebPermission
        }
        else {
            Write-Host "Site URL is empty." -ForegroundColor Red
        }
    }
    catch {
        Write-Host "Error in connecting to Site:'$($SiteUrl)'" $_.Exception.Message -ForegroundColor Red               
    } 
}

Function WebPermission {
    try {
        $Web = Get-PnPWeb -Includes RoleAssignments
        CheckPermission $Web    
        SubWebPermission        
    }
    catch {
        Write-Host "Error in getting web:" $_.Exception.Message -ForegroundColor Red               
    } 
}

Function CheckPermission ($obj) {
    try {
        Write-Host "Getting permission for the :'$($obj.Url)'..." -ForegroundColor Yellow
        Get-PnPProperty -ClientObject $obj -Property HasUniqueRoleAssignments, RoleAssignments      
        $HasUniquePermissions = $obj.HasUniqueRoleAssignments
   
        Foreach ($RoleAssignment in $obj.RoleAssignments) {                
            Get-PnPProperty -ClientObject $RoleAssignment -Property RoleDefinitionBindings, Member
                  
            $PermissionType = $RoleAssignment.Member.PrincipalType
                     
            $PermissionLevels = $RoleAssignment.RoleDefinitionBindings | Select -ExpandProperty Name
                
            If ($PermissionLevels.Length -eq 0) { Continue } 

            If ($PermissionType -eq "SharePointGroup") {
                    
                $GroupMembers = Get-PnPGroupMembers -Identity $RoleAssignment.Member.LoginName                                  
                If ($GroupMembers.count -eq 0) { Continue }
                ForEach ($User in $GroupMembers) {
                    $global:permissions += New-Object PSObject -Property ([ordered]@{
                            'Site URL'           = $obj.Url
                            'Site Title'         = $obj.Title
                            Title                = $User.Title 
                            PermissionType       = $PermissionType
                            PermissionLevels     = $PermissionLevels -join ","
                            Member               = $RoleAssignment.Member.Title     
                            HasUniquePermissions = $HasUniquePermissions                                     
                        })  
                }
            }                        
            Else {                                        
                $global:permissions += New-Object PSObject -Property ([ordered]@{
                        'Site URL'           = $obj.Url
                        'Site Title'         = $obj.Title
                        Title                = $RoleAssignment.Member.Title 
                        PermissionType       = $PermissionType
                        PermissionLevels     = $PermissionLevels -join ","
                        Member               = "Direct Permission"      
                        HasUniquePermissions = $HasUniquePermissions                             
                    })  
            }                            
        }                                  
        BindingtoCSV($global:permissions)
        $global:permissions = @()
        Write-Host "Getting permission successfully for the :'$($obj.Url)'..." -ForegroundColor Green
    }
    catch {
        Write-Host "Error in checking permission" $_.Exception.Message -ForegroundColor Red               
    } 
}

Function SubWebPermission {
    try {    
        $subwebs = Get-PnPSubWebs -Recurse  
        foreach ($subweb in $subwebs) { 
            Write-Host "Connecting to Subweb :'$($subweb.Url)'..." -ForegroundColor Yellow
            Connect-PnPOnline -Url $subweb.Url -Credentials $Creds
            Write-Host "Connection successfully to Subweb :'$($subweb.Url)'..." -ForegroundColor Green
            CheckPermission $subweb
        } 
    }
    catch {
        Write-Host "Error in connecting to sub web" $_.Exception.Message -ForegroundColor Red               
    } 
}

Function BindingtoCSV {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $Global)       
    $global:permissions | Export-Csv $CSVPath -NoTypeInformation -Append            
}

ConnectToSPSite

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL to export permissions from")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory = $false, HelpMessage = "Output CSV file path")]
    [ValidateScript({ Test-Path (Split-Path $_ -Parent) -PathType Container })]
    [string]$OutputPath = "sitepermissions_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
)

begin {
    Write-Verbose "Authenticating with Microsoft 365..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) { 
        throw "Failed to authenticate with Microsoft 365"
    }
    Write-Host "Successfully authenticated with Microsoft 365" -ForegroundColor Green
    
    $script:ReportCollection = [System.Collections.ArrayList]::new()
    
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = Join-Path ([System.IO.Path]::GetTempPath()) "ExportSitePermissions-$timestamp.log"
    Start-Transcript -Path $transcriptPath
    Write-Host "Transcript started: $transcriptPath" -ForegroundColor Cyan
    
    $script:Summary = @{
        WebsProcessed = 0
        PermissionsFound = 0
        GroupsExpanded = 0
        Failures = 0
    }
    
    function Get-WebPermissionsRecursive {
        param([string]$WebUrl)
        
        try {
            Write-Verbose "Processing web: $WebUrl"
            Write-Progress -Activity "Exporting Site Permissions" -Status "Processing: $WebUrl" -CurrentOperation "Getting web permissions"
            $script:Summary.WebsProcessed++
            
            $webJson = m365 spo web get --url $WebUrl --withPermissions --output json 2>&1 | Out-String
            if ($LASTEXITCODE -ne 0) { 
                throw "Failed to get web: $webJson"
            }
            $web = $webJson | ConvertFrom-Json
            
            if ($null -eq $web.RoleAssignments -or $web.RoleAssignments.Count -eq 0) {
                Write-Verbose "No role assignments found for web: $WebUrl"
            }
            else {
                foreach ($roleAssignment in $web.RoleAssignments) {
                    $permissionType = switch ($roleAssignment.Member.PrincipalType) {
                        1 { 'User' }
                        4 { 'SecurityGroup' }
                        8 { 'SharePointGroup' }
                        default { $roleAssignment.Member.PrincipalType }
                    }
                    
                    $permissionLevels = ($roleAssignment.RoleDefinitionBindings | Select-Object -ExpandProperty Name) -join ', '
                    
                    if ([string]::IsNullOrWhiteSpace($permissionLevels)) { 
                        Write-Verbose "Skipping role assignment with no permission levels"
                        continue 
                    }
                    
                    if ($roleAssignment.Member.PrincipalType -eq 8) {
                        Write-Verbose "Expanding SharePoint group: $($roleAssignment.Member.Title)"
                        $script:Summary.GroupsExpanded++
                        
                        try {
                            $membersJson = m365 spo group member list --webUrl $WebUrl --groupId $roleAssignment.Member.Id --output json 2>&1 | Out-String
                            if ($LASTEXITCODE -ne 0) { 
                                Write-Warning "Failed to get members for group '$($roleAssignment.Member.Title)': $membersJson"
                                continue
                            }
                            $members = @($membersJson | ConvertFrom-Json)
                            
                            if ($members.Count -eq 0) {
                                Write-Verbose "Group '$($roleAssignment.Member.Title)' has no members"
                                continue
                            }
                            
                            foreach ($member in $members) {
                                [void]$script:ReportCollection.Add([PSCustomObject]@{
                                    'Site URL' = $web.Url
                                    'Site Title' = $web.Title
                                    'Title' = $member.Title
                                    'PermissionType' = $permissionType
                                    'PermissionLevels' = $permissionLevels
                                    'Member' = $roleAssignment.Member.Title
                                    'HasUniquePermissions' = $web.HasUniqueRoleAssignments
                                })
                                $script:Summary.PermissionsFound++
                            }
                        }
                        catch {
                            Write-Warning "Error expanding group '$($roleAssignment.Member.Title)': $_"
                            continue
                        }
                    }
                    else {
                        [void]$script:ReportCollection.Add([PSCustomObject]@{
                            'Site URL' = $web.Url
                            'Site Title' = $web.Title
                            'Title' = $roleAssignment.Member.Title
                            'PermissionType' = $permissionType
                            'PermissionLevels' = $permissionLevels
                            'Member' = 'Direct Permission'
                            'HasUniquePermissions' = $web.HasUniqueRoleAssignments
                        })
                        $script:Summary.PermissionsFound++
                    }
                }
            }
            
            Write-Verbose "Getting subwebs for: $WebUrl"
            $subwebsJson = m365 spo web list --url $WebUrl --output json 2>&1 | Out-String
            if ($LASTEXITCODE -eq 0) {
                $subwebs = @($subwebsJson | ConvertFrom-Json)
                if ($subwebs.Count -gt 0) {
                    Write-Verbose "Found $($subwebs.Count) subweb(s)"
                    foreach ($subweb in $subwebs) {
                        Get-WebPermissionsRecursive -WebUrl $subweb.Url
                    }
                }
                else {
                    Write-Verbose "No subwebs found for: $WebUrl"
                }
            }
            else {
                Write-Verbose "Failed to get subwebs (may not exist): $subwebsJson"
            }
        }
        catch {
            Write-Warning "Failed to process web '$WebUrl': $_"
            $script:Summary.Failures++
        }
    }
}

process {
    Write-Host "`nStarting permission export from site: $SiteUrl" -ForegroundColor Cyan
    Get-WebPermissionsRecursive -WebUrl $SiteUrl
    Write-Progress -Activity "Exporting Site Permissions" -Completed
}

end {
    if ($script:ReportCollection.Count -gt 0) {
        try {
            $script:ReportCollection | Export-Csv -Path $OutputPath -NoTypeInformation
            Write-Host "`nExported $($script:ReportCollection.Count) permission records to: $OutputPath" -ForegroundColor Green
        }
        catch {
            Write-Host "`nFailed to export CSV: $_" -ForegroundColor Red
        }
    }
    else {
        Write-Host "`nNo permissions found to export." -ForegroundColor Yellow
    }
    
    Write-Host "`n=== Permission Export Summary ===" -ForegroundColor Cyan
    Write-Host "Webs Processed: $($script:Summary.WebsProcessed)" -ForegroundColor White
    
    $permColor = if ($script:Summary.PermissionsFound -gt 0) { "Green" } else { "Yellow" }
    Write-Host "Permissions Found: $($script:Summary.PermissionsFound)" -ForegroundColor $permColor
    
    if ($script:Summary.GroupsExpanded -gt 0) {
        Write-Host "Groups Expanded: $($script:Summary.GroupsExpanded)" -ForegroundColor Green
    }
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
    
    Write-Host "==================================`n" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Export permissions from a site collection and all subwebs
# .\Export-SitePermissions.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project"

# Export with custom output path and verbose logging
# .\Export-SitePermissions.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -OutputPath "C:\Reports\permissions.csv" -Verbose

# Export from root site collection
# .\Export-SitePermissions.ps1 -SiteUrl "https://contoso.sharepoint.com"

# Export with pipeline support (process multiple sites)
# @("https://contoso.sharepoint.com/sites/site1", "https://contoso.sharepoint.com/sites/site2") | ForEach-Object { .\Export-SitePermissions.ps1 -SiteUrl $_ }
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-sitecollection-permission-with-subwebs" aria-hidden="true" />

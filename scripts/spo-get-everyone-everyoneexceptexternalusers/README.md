

# Audit 'Everyone' and  'Everyone except external users' claim within a SharePoint site

## Summary

As part of Microsoft 365 Copilot readiness, you may want to find where "Everyone and "Everyone except external users" claims are granted permissions which is a cause of oversharing.

![Example Screenshot](assets/example.png)

### Prerequisites

- The user account that runs the script must have SharePoint Online site administrator access./

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage = "URL of a single SharePoint site to audit")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,
    
    [Parameter(HelpMessage = "Include list-level permission audit (slower)")]
    [switch]$IncludeListPermissions,
    
    [Parameter(HelpMessage = "Include list item-level permission audit (much slower, requires unique permissions per item)")]
    [switch]$IncludeListItemPermissions,
    
    [Parameter(HelpMessage = "Path where the CSV report will be saved")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login status..."
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure login to CLI for Microsoft 365. Please run 'm365 login' first."
    }
    
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath)) {
            throw "Output path '$OutputPath' does not exist. Please provide a valid directory path."
        }
    }
    
    $script:AuditCollection = @()
    $script:Summary = @{
        SitesAudited = 0
        GroupMatches = 0
        ListMatches = 0
        ItemMatches = 0
        Failures = 0
    }
    
    $everyoneClaims = @(
        "everyone except external users",
        "everyone",
        "all users",
        "spo-grid-all-users",
        "c:0(.s|true",
        "c:0-.f|rolemanager"
    )
    
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = Join-Path $OutputPath "EveryoneAudit_Transcript_$timestamp.log"
    Start-Transcript -Path $transcriptPath
}

process {
    try {
        Write-Host "\nAuditing site: $SiteUrl" -ForegroundColor Cyan
        Write-Verbose "Retrieving site with associated groups..."
        
        $webJson = m365 spo web get --url $SiteUrl --withGroups --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve site information. CLI: $webJson"
        }
        
        $web = @($webJson | ConvertFrom-Json)
        $script:Summary.SitesAudited++
        Write-Verbose "Successfully retrieved site: $($web.Title)"
        
        $associatedGroups = @()
        if ($web.AssociatedOwnerGroup) {
            $associatedGroups += [PSCustomObject]@{
                Id = $web.AssociatedOwnerGroup.Id
                Title = $web.AssociatedOwnerGroup.Title
                Type = "Owner"
            }
        }
        if ($web.AssociatedMemberGroup) {
            $associatedGroups += [PSCustomObject]@{
                Id = $web.AssociatedMemberGroup.Id
                Title = $web.AssociatedMemberGroup.Title
                Type = "Member"
            }
        }
        if ($web.AssociatedVisitorGroup) {
            $associatedGroups += [PSCustomObject]@{
                Id = $web.AssociatedVisitorGroup.Id
                Title = $web.AssociatedVisitorGroup.Title
                Type = "Visitor"
            }
        }
        
        Write-Host "  Auditing $($associatedGroups.Count) associated groups..." -ForegroundColor White
        
        foreach ($group in $associatedGroups) {
            try {
                Write-Verbose "Checking members of group: $($group.Title) (ID: $($group.Id))"
                
                $membersJson = m365 spo group member list --webUrl $SiteUrl --groupId $group.Id --output json
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve members for group '$($group.Title)'. CLI: $membersJson"
                    $script:Summary.Failures++
                    continue
                }
                
                $members = @($membersJson | ConvertFrom-Json)
                
                foreach ($member in $members) {
                    $loginName = $member.LoginName.ToLower()
                    $matchedClaim = $everyoneClaims | Where-Object { $loginName -like "*$_*" }
                    
                    if ($matchedClaim) {
                        $script:AuditCollection += [PSCustomObject]@{
                            SiteUrl = $SiteUrl
                            SiteTitle = $web.Title
                            ListTitle = ""
                            Type = "Group"
                            RelativeUrl = ""
                            ParentGroup = $group.Title
                            ParentGroupType = $group.Type
                            MemberType = "Claim"
                            MemberName = $member.Title
                            MemberLoginName = $member.LoginName
                            Roles = ""
                        }
                        
                        $script:Summary.GroupMatches++
                        Write-Host "    Found 'Everyone' claim in $($group.Type) group: $($member.Title)" -ForegroundColor Yellow
                    }
                }
            }
            catch {
                Write-Warning "Error auditing group '$($group.Title)': $($_.Exception.Message)"
                $script:Summary.Failures++
                continue
            }
        }
        
        if ($IncludeListPermissions) {
            Write-Host "  Auditing document libraries for unique permissions..." -ForegroundColor White
            
            try {
                Write-Verbose "Retrieving document libraries with HasUniqueRoleAssignments property..."
                $listsJson = m365 spo list list --webUrl $SiteUrl --properties "Id,Title,Hidden,BaseTemplate,HasUniqueRoleAssignments,RootFolder/ServerRelativeUrl" --output json
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve document libraries. CLI: $listsJson"
                    $script:Summary.Failures++
                    continue
                }
                
                $lists = @($listsJson | ConvertFrom-Json)
                $filteredLists = $lists | Where-Object { $_.Hidden -eq $false -and $_.BaseTemplate -eq 101 -and $_.Title -notin @('Form Templates','Style Library','Site Pages') }
                
                Write-Verbose "Total document libraries: $($lists.Count), After filtering: $($filteredLists.Count)"
                
                $listsWithUniquePerms = $filteredLists | Where-Object { $_.HasUniqueRoleAssignments -eq $true }
                Write-Verbose "Libraries with unique permissions: $($listsWithUniquePerms.Count)"
                
                if ($listsWithUniquePerms.Count -eq 0) {
                    Write-Verbose "No document libraries with unique permissions found."
                    continue
                }
                
                foreach ($list in $listsWithUniquePerms) {
                    try {
                        Write-Verbose "Auditing list: $($list.Title)"
                        
                        $listWithPermsJson = m365 spo list get --webUrl $SiteUrl --id $list.Id --withPermissions --output json
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to retrieve permissions for list '$($list.Title)'. CLI: $listWithPermsJson"
                            $script:Summary.Failures++
                            continue
                        }
                        
                        $listWithPerms = @($listWithPermsJson | ConvertFrom-Json)
                        
                        if ($listWithPerms.RoleAssignments) {
                            foreach ($assignment in $listWithPerms.RoleAssignments) {
                                $memberLoginName = $assignment.Member.LoginName.ToLower()
                                $matchedClaim = $everyoneClaims | Where-Object { $memberLoginName -like "*$_*" }
                                
                                if ($matchedClaim) {
                                    $roles = ($assignment.RoleDefinitionBindings | ForEach-Object { $_.Name }) -join ', '
                                    
                                    $script:AuditCollection += [PSCustomObject]@{
                                        SiteUrl = $SiteUrl
                                        SiteTitle = $web.Title
                                        ListTitle = $list.Title
                                        Type = "List"
                                        RelativeUrl = $list.RootFolder.ServerRelativeUrl
                                        ParentGroup = ""
                                        ParentGroupType = ""
                                        MemberType = "Claim"
                                        MemberName = $assignment.Member.Title
                                        MemberLoginName = $assignment.Member.LoginName
                                        Roles = $roles
                                    }
                                    
                                    $script:Summary.ListMatches++
                                    Write-Host "    Found 'Everyone' claim in list '$($list.Title)': $($assignment.Member.Title)" -ForegroundColor Yellow
                                }
                            }
                        }
                        
                        if ($IncludeListItemPermissions) {
                            Write-Verbose "Auditing items with unique permissions in list: $($list.Title)"
                            
                            try {
                                $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listId $list.Id --fields "Id,FileRef,FileLeafRef,FSObjType,HasUniqueRoleAssignments" --filter "FSObjType eq 0 and HasUniqueRoleAssignments eq true" --output json
                                if ($LASTEXITCODE -ne 0) {
                                    Write-Warning "Failed to retrieve items with unique permissions from list '$($list.Title)'. CLI: $itemsJson"
                                    $script:Summary.Failures++
                                }
                                else {
                                    $itemsWithUniquePerms = @($itemsJson | ConvertFrom-Json)
                                    
                                    if ($itemsWithUniquePerms.Count -gt 0) {
                                        Write-Verbose "Found $($itemsWithUniquePerms.Count) items with unique permissions in list '$($list.Title)'"
                                        
                                        foreach ($item in $itemsWithUniquePerms) {
                                            try {
                                                Write-Verbose "Auditing item: $($item.FileLeafRef) (ID: $($item.Id))"
                                                
                                                $itemRoleAssignmentsJson = m365 request --url "$SiteUrl/_api/web/lists(guid'$($list.Id)')/items($($item.Id))/RoleAssignments?`$expand=Member,RoleDefinitionBindings" --method GET --output json
                                                if ($LASTEXITCODE -ne 0) {
                                                    Write-Warning "Failed to retrieve role assignments for item '$($item.FileLeafRef)'. CLI: $itemRoleAssignmentsJson"
                                                    $script:Summary.Failures++
                                                }
                                                else {
                                                    $itemRoleAssignments = @($itemRoleAssignmentsJson | ConvertFrom-Json)
                                                    
                                                    if ($itemRoleAssignments.value -and $itemRoleAssignments.value.Count -gt 0) {
                                                        foreach ($assignment in $itemRoleAssignments.value) {
                                                            if ($assignment.Member -and $assignment.Member.LoginName) {
                                                                $loginName = $assignment.Member.LoginName.ToLower()
                                                                $matchedClaim = $everyoneClaims | Where-Object { $loginName -like "*$_*" }
                                                                
                                                                if ($matchedClaim) {
                                                                    $roles = ($assignment.RoleDefinitionBindings | Select-Object -ExpandProperty Name) -join '|'
                                                                    
                                                                    $script:AuditCollection += [PSCustomObject]@{
                                                                        SiteUrl = $SiteUrl
                                                                        SiteTitle = $web.Title
                                                                        ListTitle = $list.Title
                                                                        Type = "Item"
                                                                        RelativeUrl = $item.FileRef
                                                                        ParentGroup = ""
                                                                        ParentGroupType = ""
                                                                        MemberType = "Claim"
                                                                        MemberName = $assignment.Member.Title
                                                                        MemberLoginName = $assignment.Member.LoginName
                                                                        Roles = $roles
                                                                    }
                                                                    
                                                                    $script:Summary.ItemMatches++
                                                                    Write-Host "        Found 'Everyone' claim on item '$($item.FileLeafRef)': $($assignment.Member.Title)" -ForegroundColor Yellow
                                                                }
                                                            }
                                                        }
                                                    }
                                                }
                                            }
                                            catch {
                                                Write-Warning "Error auditing item '$($item.FileLeafRef)': $($_.Exception.Message)"
                                                $script:Summary.Failures++
                                                continue
                                            }
                                        }
                                    }
                                    else {
                                        Write-Verbose "No items with unique permissions found in list '$($list.Title)'"
                                    }
                                }
                            }
                            catch {
                                Write-Warning "Error retrieving items from list '$($list.Title)': $($_.Exception.Message)"
                                $script:Summary.Failures++
                                continue
                            }
                        }
                    }
                    catch {
                        Write-Warning "Error auditing list '$($list.Title)': $($_.Exception.Message)"
                        $script:Summary.Failures++
                        continue
                    }
                }
            }
        }
    }
    catch {
        Write-Warning "Failed to audit site '$SiteUrl': $($_.Exception.Message)"
        $script:Summary.Failures++
    }
}

end {
    $csvPath = Join-Path $OutputPath "EveryoneAudit_$timestamp.csv"
    
    if ($script:AuditCollection.Count -gt 0) {
        $script:AuditCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "\nExported report to: $csvPath" -ForegroundColor Green
    }
    else {
        Write-Host "\nNo 'Everyone' claims found in audited site." -ForegroundColor Green
    }
    
    Write-Host "\n========== Audit Summary ==========" -ForegroundColor White
    Write-Host "Sites Audited   : $($script:Summary.SitesAudited)" -ForegroundColor White
    Write-Host "Group Matches   : $($script:Summary.GroupMatches)" -ForegroundColor White
    Write-Host "List Matches    : $($script:Summary.ListMatches)" -ForegroundColor White
    Write-Host "Item Matches    : $($script:Summary.ItemMatches)" -ForegroundColor White
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures        : $($script:Summary.Failures)" -ForegroundColor Red
    }
    else {
        Write-Host "Failures        : 0" -ForegroundColor Green
    }
    
    Stop-Transcript
}

# Usage examples:
#
# Example 1: Audit a single site (groups only, fast)
# .\spo-get-everyone-everyoneexceptexternalusers.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project"
#
# Example 2: Audit with list-level permissions (slower)
# .\spo-get-everyone-everyoneexceptexternalusers.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -IncludeListPermissions
#
# Example 3: Comprehensive audit with list and item-level permissions (much slower)
# .\spo-get-everyone-everyoneexceptexternalusers.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -IncludeListPermissions -IncludeListItemPermissions
#
# Example 4: Specify custom output path with verbose logging
# .\spo-get-everyone-everyoneexceptexternalusers.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -IncludeListPermissions -OutputPath "C:\Reports" -Verbose
```

# [PnP PowerShell](#tab/pnpps)

```powershell
param (
    [Parameter(Mandatory = $true)]
    [string] $domain
)
Clear-Host

$properties=@{SiteUrl='';SiteTitle='';ListTitle='';SensitivityLabel='';Type='';RelativeUrl='';ParentGroup='';MemberType='';MemberName='';MemberLoginName='';Roles='';}; 

$adminSiteURL = "https://$domain-Admin.SharePoint.com"
$TenantURL = "https://$domain.sharepoint.com" 
$dateTime = (Get-Date).toString("dd-MM-yyyy-hh-ss")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = (Split-Path $invocation.MyCommand.Path) + "\Logs\"
$excludeLimitedAccess = $true;
$includeListsItems = $true;

#$SiteCollectionUrl = Read-Host -Prompt "Enter site collection URL ";
$everyoneGroups = @("everyone except external users", "everyone","all users")

$global:siteTitle= "";
#Exclude certain libraries
$ExcludedLibraries = @("Form Templates", "Preservation Hold Library", "Site Assets", "Images", "Pages", "Settings", "Videos","Timesheet"
  "Site Collection Documents", "Site Collection Images", "Style Library", "AppPages", "Apps for SharePoint", "Apps for Office")

$global:permissions =@();
$global:sharingLinks = @();

function Get-ListItems_WithUniquePermissions{
  param(
      [Parameter(Mandatory)]
      [Microsoft.SharePoint.Client.List]$List
  )
  $selectFields = "ID,HasUniqueRoleAssignments,FileRef,FileLeafRef,FileSystemObjectType"
 
  $Url = $siteUrl + '/_api/web/lists/getbytitle(''' + $($list.Title) + ''')/items?$select=' + $($selectFields)
  $nextLink = $Url
  $listItems = @()
  $Stoploop =$true
  while($nextLink){  
      do{
      try {
          $response = invoke-pnpsprestmethod -Url $nextLink -Method Get
          $Stoploop =$true
  
      }
      catch {
          write-host "An error occured: $_  : Retrying" -ForegroundColor Red
          $Stoploop =$true
          Start-Sleep -Seconds 30
      }
  }
  While ($Stoploop -eq $false)
  
      $listItems += $response.value | where-object{$_.HasUniqueRoleAssignments -eq $true}
      if($response.'odata.nextlink'){
          $nextLink = $response.'odata.nextlink'
      }    else{
          $nextLink = $null
      }
  }

  return $listItems
}

Function PermissionObject($_object,$_type,$_relativeUrl,$_siteUrl,$_siteTitle,$_listTitle,$_memberType,$_parentGroup,$_memberName,$_memberLoginName,$_roleDefinitionBindings,$_sensitivityLabel)
{
  $permission = New-Object -TypeName PSObject -Property $properties; 
  $permission.SiteUrl =$_siteUrl; 
  $permission.SiteTitle = $_siteTitle; 
  $permission.ListTitle = $_listTitle; 
  $permission.SensitivityLabel = $_sensitivityLabel; 
  $permission.Type =  $_Type -eq 1 ? "Folder" : $_Type -eq 0 ? "File" : $_Type;
  $permission.RelativeUrl = $_relativeUrl; 
  $permission.MemberType = $_memberType; 
  $permission.ParentGroup = $_parentGroup; 
  $permission.MemberName = $_memberName; 
  $permission.MemberLoginName = $_memberLoginName; 
  $permission.Roles = $_roleDefinitionBindings -join ","; 
  $global:permissions += $permission;
}

Function Extract-Guid ($inputString) {
  $splitString = $inputString -split '\|'
  return $splitString[2].TrimEnd('_o')
}

Function QueryUniquePermissionsByObject($_ctx,$_object,$_Type,$_RelativeUrl,$_siteUrl,$_siteTitle,$_listTitle)
{
  $roleAssignments = Get-PnPProperty -ClientObject $_object -Property RoleAssignments
   switch ($_Type) {
    0 { $sensitivityLabel = $_object.FieldValues["_DisplayName"] }
    1 { $sensitivityLabel = $_object.FieldValues["_DisplayName"] }
    "Site" { $sensitivityLabel = (Get-PnPSiteSensitivityLabel).displayname }
    default { " " }
}
  foreach($roleAssign in $roleAssignments){
    Get-PnPProperty -ClientObject $roleAssign -Property RoleDefinitionBindings,Member;
    $PermissionLevels = $roleAssign.RoleDefinitionBindings | Select -ExpandProperty Name;
    #Get all permission levels assigned (Excluding:Limited Access)  
    if($excludeLimitedAccess -eq $true){
       $PermissionLevels = ($PermissionLevels | Where { $_ -ne "Limited Access"}) -join ","  
    }

    $MemberType = $roleAssign.Member.GetType().Name; 
    #Get the Principal Type: User, SP Group, AD Group  
    $PermissionType = $roleAssign.Member.PrincipalType  

    If($PermissionLevels.Length -gt 0) {
      $MemberType = $roleAssign.Member.GetType().Name; 
       #Ignoring sharing links as sharing links are not supported for everyone group
       
      If($roleAssign.Member.Title -notlike "SharingLinks*" -and ($MemberType -eq "Group" -or $MemberType -eq "User"))
      { 
        $MemberName = $roleAssign.Member.Title; 
        $groupExists = $false;
        $everyonegroups | ForEach-Object {if($roleAssign.Member.Title -contains $_){$groupExists =$true}}
        $MemberLoginName = $roleAssign.Member.LoginName;    
        if($groupExists -eq $true){
         
        if($MemberType -eq "User")
        {
          $ParentGroup = "NA";
        }
        else
        {
          $ParentGroup = $MemberName;
        }
        (PermissionObject $_object $_Type $_RelativeUrl $_siteUrl $_siteTitle $_listTitle $MemberType $ParentGroup $MemberName $MemberLoginName $PermissionLevels $sensitivityLabel); 
        }  
    }

      if($_Type  -eq "Site" -and $MemberType -eq "Group")
      {
        $sensitivityLabel = (Get-PnPSiteSensitivityLabel).DisplayName
        If($PermissionType -eq "SharePointGroup")  {  
          #Get Group Members  
          $groupUsers = Get-PnPGroupMember -Identity $roleAssign.Member.LoginName                  
          $groupUsers|foreach-object{ 
            $groupExists = $false;
            $title = $_.Title
            $everyonegroups | ForEach-Object {if($title -contains $_){$groupExists =$true}}
            if($groupExists -eq $true){
            (PermissionObject $_object "Site" $_RelativeUrl $_siteUrl $_siteTitle "" "GroupMember" $roleAssign.Member.LoginName $_.Title $_.LoginName $PermissionLevels $sensitivityLabel);   
            }
        }
        }
      } 
    }      
  }
}
Function QueryUniquePermissions($_web)
{
  ##query list, files and items unique permissions
  Write-Host "Querying web $($_web.Title)";
  $siteUrl = $_web.Url; 
 
  Write-Host $siteUrl -Foregroundcolor "Red"; 
  $global:siteTitle = $_web.Title; 
  $ll = Get-PnPList -Includes BaseType, Hidden, Title,HasUniqueRoleAssignments,RootFolder  -Connection $siteconn | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLibraries } #$_.BaseType -eq "DocumentLibrary" 
  Write-Host "Number of lists $($ll.Count)";

  QueryUniquePermissionsByObject $_web $_web "Site" "" $siteUrl $siteTitle  "";
 
  foreach($list in $ll)
  {      
    $listUrl = $list.RootFolder.ServerRelativeUrl; 
    #Exclude internal system lists and check if it has unique permissions 
    if($list.Hidden -ne $True)
    { 
      Write-Host $list.Title  -Foregroundcolor "Yellow"; 
      $listTitle = $list.Title; 
      #Check List Permissions 
      if($list.HasUniqueRoleAssignments -eq $True)
      { 
        $Type = $list.BaseType.ToString(); 
        QueryUniquePermissionsByObject $_web $list $Type $listUrl $siteUrl $siteTitle $listTitle;
      }
      
      if($includeListsItems){         
        $collListItem =  Get-ListItems_WithUniquePermissions -List $list
        $count = $collListItem.Count
        Write-Host  "Number of items with unique permissions: $count within list $listTitle" 
        foreach($item in $collListItem) 
        {
            $Type = $item.FileSystemObjectType; 
            $fileUrl = $item.FileRef;  
            $i = Get-PnPListItem -List $list -Id $item.ID
            QueryUniquePermissionsByObject $_web $i $Type $fileUrl $siteUrl $siteTitle $listTitle;
        } 
      }
    }
  }
}

if(Test-Path $directorypath){
  Connect-PnPOnline -Url $adminSiteURL -Interactive 

  $adminConnection = Get-PnPConnection
  Get-PnPTenantSite -Filter "Url -like '$TenantURL'" -Connection $adminConnection | Where-Object { $_.Template -ne 'RedirectSite#0' }  | foreach-object {   
    Write-Host "Processing Site:" $_.Url -ForegroundColor Magenta

  Connect-PnPOnline -Url $_.Url -Interactive

   #array storing permissions
   $web = Get-PnPWeb
   #root web , i.e. site collection level
   QueryUniquePermissions($web);
}
  Write-Host "Permission count: $($global:permissions.Count)";
  $exportFilePath = Join-Path -Path $directorypath -ChildPath $([string]::Concat($domain,"-everyone_",$dateTime,".csv"));
  
  Write-Host "Export File Path is:" $exportFilePath
  Write-Host "Number of lines exported is :" $global:permissions.Count
 
  $global:permissions | Select-Object SiteUrl,SiteTitle,Type,SensitivityLabel,RelativeUrl,ListTitle,MemberType,MemberName,MemberLoginName,ParentGroup,Roles|Export-CSV -Path $exportFilePath -NoTypeInformation;
  
}
else{
  Write-Host "Invalid directory path:" $directorypath -ForegroundColor "Red";
}
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Manage 'Everyone' and 'Everyone except external users' claim within a SharePoint site using PowerShell](https://reshmeeauckloo.com/posts/powershell-get-everyone-report-sharepoint/)

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Reshmee Auckloo](https://github.com/reshmee011) |
| TiloGit |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-everyone-everyoneexceptexternalusers" aria-hidden="true" />

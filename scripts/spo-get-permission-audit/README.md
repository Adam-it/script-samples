

# Get permissions including unique permissions up to item level including sharing links

## Summary

Managing permissions in SharePoint is a critical aspect of maintaining data security and compliance within organisations. However, as SharePoint environments grow in complexity, manually auditing and managing permissions becomes increasingly challenging. This script is available for both PnP PowerShell and CLI for Microsoft 365 v11.4.0+.

Copilot for Microsoft m365 can access data from all the tenant, whether it's Outlook emails, Teams chats and meetings, SharePoint and OneDrive. SharePoint is where all most documents, videos, and more are stored. Hence permission audit across sensitive sites to ensure "Least privilege" is a must to avoid data leak while using Copilot for Microsoft m365 which makes it easier to discover content through prompts.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have SharePoint Online site administrator access.

# [PnP PowerShell](#tab/pnpps)

```powershell

Clear-Host

$properties=@{SiteUrl='';SiteTitle='';ListTitle='';SensitivityLabel='';Type='';RelativeUrl='';ParentGroup='';MemberType='';MemberName='';MemberLoginName='';Roles='';}; 
 
$dateTime = (Get-Date).toString("dd-MM-yyyy-hh-ss")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = (Split-Path $invocation.MyCommand.Path) + "\"
$excludeLimitedAccess = $true;
$includeListsItems = $true;

$SiteCollectionUrl = Read-Host -Prompt "Enter site collection URL ";
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
    $Users = Get-PnPProperty -ClientObject ($roleAssign.Member) -Property Users -ErrorAction SilentlyContinue
    #Get Access type
    $AccessType = $roleAssign.RoleDefinitionBindings.Name
    $MemberType = $roleAssign.Member.GetType().Name; 
    #Get the Principal Type: User, SP Group, AD Group  
    $PermissionType = $roleAssign.Member.PrincipalType  
  if( $_Type -eq 0){
      $sharingLinks = Get-PnPFileSharingLink -Identity $_object.FieldValues["FileRef"]
  }
  if( $_Type -eq 1){
      $sharingLinks = Get-PnPFolderSharingLink -Folder $_object.FieldValues["FileRef"]
  }

    If($PermissionLevels.Length -gt 0) {
      $MemberType = $roleAssign.Member.GetType().Name; 
       #Sharing link is in the format SharingLinks.03012675-2057-4d1d-91e0-8e3b176edd94.OrganizationView.20d346d3-d359-453b-900c-633c1551ccaa
        If ($roleAssign.Member.Title -like "SharingLinks*")
        {
          if($sharingLinks){
          $sharingLinks | where-object {$roleAssign.Member.Title -match $_.Id } | ForEach-Object{
            If ($Users.Count -gt 0) 
            {
                ForEach ($User in $Users)
                {
                PermissionObject $_object $_Type $_RelativeUrl $_siteUrl $_siteTitle $_listTitle "Sharing Links" $roleAssign.Member.LoginName  $user.Title $User.LoginName $_.Link.Type $sensitivityLabel; 
                }
            } 
            else {
              PermissionObject $_object $_Type $_RelativeUrl $_siteUrl $_siteTitle $_listTitle "Sharing Links" $roleAssign.Member.LoginName  $_.Link.Scope "" $_.Link.Type  $sensitivityLabel;
            }
          }  
        }
        <#  
        If ($Users.Count -gt 0) 
            {
                ForEach ($User in $Users)
                {
                PermissionObject $_object $_Type $_RelativeUrl $_siteUrl $_siteTitle $_listTitle "Sharing Links" $roleAssign.Member.LoginName  $user.Title $User.LoginName $AccessType $sensitivityLabel; 
                }
            } 
            else{
              if($sharingLinks){
                $sharingLinks | where-object {$roleAssign.Member.Title -match $_.Id } | ForEach-Object{
                  PermissionObject $_object $_Type $_RelativeUrl $_siteUrl $_siteTitle $_listTitle "Sharing Links" $roleAssign.Member.Title  $_.Link.Scope "" $_.Link.Type $sensitivityLabel;
                }
              }
              else{
                #find whether the sharing link is organisation or anyone
                PermissionObject $_object $_Type $_RelativeUrl $_siteUrl $_siteTitle $_listTitle "Sharing Links" $roleAssign.Member.Title  "All"  $roleAssign.Member.Title $roleAssign.RoleDefinitionBindings.Description $sensitivityLabel;
              }
            }#>
        }
      ElseIf($MemberType -eq "Group" -or $MemberType -eq "User")
      { 
        $MemberName = $roleAssign.Member.Title; 
        $MemberLoginName = $roleAssign.Member.LoginName;    
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

      if($_Type  -eq "Site" -and $MemberType -eq "Group")
      {
        $sensitivityLabel = (Get-PnPSiteSensitivityLabel).DisplayName
        If($PermissionType -eq "SharePointGroup")  {  
          #Get Group Members  
          $groupUsers = Get-PnPGroupMember -Identity $roleAssign.Member.LoginName                  
          $groupUsers|foreach-object{ 
            if ($_.LoginName.StartsWith("c:0o.c|federateddirectoryclaimprovider|") -and $_.LoginName.EndsWith("_0")) {
              $guid = Extract-Guid $_.LoginName
              
              Get-PnPMicrosoft365GroupOwners -Identity $guid | ForEach-Object {
                $user = $_
                (PermissionObject $_object "Site" $_RelativeUrl $_siteUrl $_siteTitle "" "GroupMember" $roleAssign.Member.LoginName $user.DisplayName $user.UserPrincipalName $PermissionLevels $sensitivityLabel); 
              }
            }
            elseif ($_.LoginName.StartsWith("c:0o.c|federateddirectoryclaimprovider|")) {
              $guid = Extract-Guid $_.LoginName
              
              Get-PnPMicrosoft365GroupMembers -Identity $guid | ForEach-Object {
                $user = $_
                (PermissionObject $_object "Site" $_RelativeUrl $_siteUrl $_siteTitle "" "GroupMember" $roleAssign.Member.LoginName $user.DisplayName $user.UserPrincipalName $PermissionLevels $sensitivityLabel); 
              }
            }

            (PermissionObject $_object "Site" $_RelativeUrl $_siteUrl $_siteTitle "" "GroupMember" $roleAssign.Member.LoginName $_.Title $_.LoginName $PermissionLevels $sensitivityLabel);   
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
 
  Connect-PnPOnline -Url $SiteCollectionUrl -Interactive
  #array storing permissions
  $web = Get-PnPWeb
  #root web , i.e. site collection level
  QueryUniquePermissions($web);

  Write-Host "Permission count: $($global:permissions.Count)";
  $exportFilePath = Join-Path -Path $directorypath -ChildPath $([string]::Concat($siteTitle,"-Permissions_",$dateTime,".csv"));
  
  Write-Host "Export File Path is:" $exportFilePath
  Write-Host "Number of lines exported is :" $global:permissions.Count
 
  $global:permissions | Select-Object SiteUrl,SiteTitle,Type,SensitivityLabel,RelativeUrl,ListTitle,MemberType,MemberName,MemberLoginName,ParentGroup,Roles|Export-CSV -Path $exportFilePath -NoTypeInformation;
  
}
else{
  Write-Host "Invalid directory path:" $directorypath -ForegroundColor "Red";
}
```

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site collection URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory = $false, HelpMessage = "Output folder path for CSV export")]
    [string]$OutputPath = (Get-Location).Path,
    
    [Parameter(Mandatory = $false, HelpMessage = "Include list item-level permissions audit")]
    [switch]$IncludeListItems,
    
    [Parameter(Mandatory = $false, HelpMessage = "Exclude Limited Access role from report")]
    [switch]$ExcludeLimitedAccess,
    
    [Parameter(Mandatory = $false, HelpMessage = "Excluded library titles")]
    [string[]]$ExcludedLibraries = @("Form Templates", "Preservation Hold Library", "Site Assets", "Images", "Pages", "Settings", "Videos", "Timesheet", "Site Collection Documents", "Site Collection Images", "Style Library", "AppPages", "Apps for SharePoint", "Apps for Office")
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    
    $script:PermissionsReport = [System.Collections.ArrayList]::new()
    $script:Summary = @{
        SitesAudited = 0
        ListsAudited = 0
        ItemsAudited = 0
        SharingLinksFound = 0
        Failures = 0
    }
    
    $logPath = Join-Path $OutputPath "PermissionAudit_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    Start-Transcript -Path $logPath
    
    Write-Host "Starting permission audit for site: $SiteUrl" -ForegroundColor Cyan
}

process {
    try {
        # LEVEL 1: Site-level permissions
        Write-Host "\nAuditing site-level permissions..." -ForegroundColor Yellow
        $webJson = m365 spo web get --url $SiteUrl --withPermissions --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve site permissions"
        }
        $web = $webJson | ConvertFrom-Json
        
        # Extract site title and sensitivity label
        $siteTitle = $web.Title
        $sensitivityLabel = ""
        
        # Try to get sensitivity label from Graph API
        try {
            Write-Verbose "  Attempting to retrieve sensitivity label..."
            $siteIdJson = m365 request --url "https://graph.microsoft.com/v1.0/sites/$($web.Url.Replace('https://', '').Replace('/', ','))" --output json
            if ($LASTEXITCODE -eq 0) {
                $siteData = $siteIdJson | ConvertFrom-Json
                if ($siteData.sensitivityLabel) {
                    $sensitivityLabel = $siteData.sensitivityLabel.displayName
                }
            }
        }
        catch {
            Write-Verbose "  Could not retrieve sensitivity label: $_"
        }
        
        Write-Verbose "  Processing site permissions for: $siteTitle"
        
        # Process site RoleAssignments
        if ($web.RoleAssignments) {
            foreach ($roleAssignment in $web.RoleAssignments) {
                $member = $roleAssignment.Member
                $roles = ($roleAssignment.RoleDefinitionBindings | ForEach-Object { $_.Name }) -join ","
                
                # Filter out Limited Access if requested
                if ($ExcludeLimitedAccess -and $roles -eq "Limited Access") {
                    continue
                }
                
                # Determine member type
                $memberType = switch ($member.PrincipalType) {
                    1 { "User" }
                    4 { "SecurityGroup" }
                    8 { "SharePointGroup" }
                    default { "Unknown" }
                }
                
                # Handle sharing links
                if ($member.Title -like "SharingLinks*") {
                    [void]$script:PermissionsReport.Add([PSCustomObject]@{
                        SiteUrl = $SiteUrl
                        SiteTitle = $siteTitle
                        Type = "Site"
                        SensitivityLabel = $sensitivityLabel
                        RelativeUrl = ""
                        ListTitle = ""
                        MemberType = "Sharing Link"
                        MemberName = $member.Title
                        MemberLoginName = $member.LoginName
                        ParentGroup = ""
                        Roles = "Sharing Link"
                    })
                    $script:Summary.SharingLinksFound++
                    continue
                }
                
                # Handle SharePoint groups - expand members
                if ($memberType -eq "SharePointGroup") {
                    try {
                        $groupMembersJson = m365 spo group member list --webUrl $SiteUrl --groupName $member.Title --output json
                        if ($LASTEXITCODE -eq 0) {
                            $groupMembers = @($groupMembersJson | ConvertFrom-Json)
                            
                            foreach ($groupMember in $groupMembers) {
                                # Check if member is M365 group (LoginName starts with c:0o.c|federateddirectoryclaimprovider|)
                                if ($groupMember.LoginName -match '^c:0o\\.c\\|federateddirectoryclaimprovider\\|(.+?)(_[om])?$') {
                                    $m365GroupId = $matches[1]
                                    
                                    try {
                                        $m365UsersJson = m365 entra m365group user list --groupId $m365GroupId --output json
                                        if ($LASTEXITCODE -eq 0) {
                                            $m365Users = @($m365UsersJson | ConvertFrom-Json)
                                            
                                            foreach ($m365User in $m365Users) {
                                                [void]$script:PermissionsReport.Add([PSCustomObject]@{
                                                    SiteUrl = $SiteUrl
                                                    SiteTitle = $siteTitle
                                                    Type = "Site"
                                                    SensitivityLabel = $sensitivityLabel
                                                    RelativeUrl = ""
                                                    ListTitle = ""
                                                    MemberType = "GroupMember"
                                                    MemberName = $m365User.displayName
                                                    MemberLoginName = $m365User.userPrincipalName
                                                    ParentGroup = $member.Title
                                                    Roles = $roles
                                                })
                                            }
                                        }
                                    }
                                    catch {
                                        Write-Verbose "  Could not expand M365 group: $m365GroupId"
                                    }
                                }
                                else {
                                    # Regular user or group
                                    [void]$script:PermissionsReport.Add([PSCustomObject]@{
                                        SiteUrl = $SiteUrl
                                        SiteTitle = $siteTitle
                                        Type = "Site"
                                        SensitivityLabel = $sensitivityLabel
                                        RelativeUrl = ""
                                        ListTitle = ""
                                        MemberType = "GroupMember"
                                        MemberName = $groupMember.Title
                                        MemberLoginName = $groupMember.LoginName
                                        ParentGroup = $member.Title
                                        Roles = $roles
                                    })
                                }
                            }
                        }
                    }
                    catch {
                        Write-Warning "  Failed to get members for group: $($member.Title)"
                        $script:Summary.Failures++
                    }
                }
                else {
                    # Direct user or security group
                    [void]$script:PermissionsReport.Add([PSCustomObject]@{
                        SiteUrl = $SiteUrl
                        SiteTitle = $siteTitle
                        Type = "Site"
                        SensitivityLabel = $sensitivityLabel
                        RelativeUrl = ""
                        ListTitle = ""
                        MemberType = $memberType
                        MemberName = $member.Title
                        MemberLoginName = $member.LoginName
                        ParentGroup = "NA"
                        Roles = $roles
                    })
                }
            }
        }
        
        $script:Summary.SitesAudited++
        
        # LEVEL 2: List-level permissions
        Write-Host "\nAuditing list-level permissions..." -ForegroundColor Yellow
        $listsJson = m365 spo list list --webUrl $SiteUrl --output json --query "[?Hidden == \`false\`]"
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve lists"
        }
        $lists = @($listsJson | ConvertFrom-Json)
        
        $filteredLists = $lists | Where-Object { $_.Title -notin $ExcludedLibraries }
        Write-Host "  Found $($filteredLists.Count) lists to audit (excluded $($lists.Count - $filteredLists.Count) system libraries)" -ForegroundColor Green
        
        $listIndex = 0
        foreach ($list in $filteredLists) {
            $listIndex++
            Write-Progress -Activity "Auditing Lists" -Status "$($list.Title) ($listIndex of $($filteredLists.Count))" -PercentComplete (($listIndex / $filteredLists.Count) * 100)
            Write-Verbose "  Processing list: $($list.Title)"
            
            try {
                # Check if list has unique permissions
                $listDetailsJson = m365 spo list get --webUrl $SiteUrl --title $list.Title --output json
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "  Failed to get list details: $($list.Title)"
                    $script:Summary.Failures++
                    continue
                }
                $listDetails = $listDetailsJson | ConvertFrom-Json
                
                if ($listDetails.HasUniqueRoleAssignments) {
                    Write-Verbose "    List has unique permissions, retrieving role assignments..."
                    
                    # Get list RoleAssignments via REST API
                    $listUrl = $list.RootFolder.ServerRelativeUrl
                    $roleAssignmentsJson = m365 request --url "$SiteUrl/_api/web/lists/getbytitle('$($list.Title)')/roleassignments?\`$expand=Member,RoleDefinitionBindings" --output json
                    if ($LASTEXITCODE -eq 0) {
                        $roleAssignmentsData = $roleAssignmentsJson | ConvertFrom-Json
                        $listRoleAssignments = @($roleAssignmentsData.value)
                        
                        foreach ($roleAssignment in $listRoleAssignments) {
                            $member = $roleAssignment.Member
                            $roles = ($roleAssignment.RoleDefinitionBindings | ForEach-Object { $_.Name }) -join ","
                            
                            if ($ExcludeLimitedAccess -and $roles -eq "Limited Access") {
                                continue
                            }
                            
                            $memberType = switch ($member.PrincipalType) {
                                1 { "User" }
                                4 { "SecurityGroup" }
                                8 { "SharePointGroup" }
                                default { "Unknown" }
                            }
                            
                            [void]$script:PermissionsReport.Add([PSCustomObject]@{
                                SiteUrl = $SiteUrl
                                SiteTitle = $siteTitle
                                Type = "List"
                                SensitivityLabel = $sensitivityLabel
                                RelativeUrl = $listUrl
                                ListTitle = $list.Title
                                MemberType = $memberType
                                MemberName = $member.Title
                                MemberLoginName = $member.LoginName
                                ParentGroup = if ($memberType -eq "User") { "NA" } else { $member.Title }
                                Roles = $roles
                            })
                        }
                    }
                    
                    $script:Summary.ListsAudited++
                }
                
                # LEVEL 3: Item-level permissions (optional)
                if ($IncludeListItems) {
                    Write-Verbose "    Checking for items with unique permissions..."
                    
                    # Get items with HasUniqueRoleAssignments = true (CRITICAL OPTIMIZATION)
                    $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listTitle $list.Title --fields "ID,FileRef,FileLeafRef,FileSystemObjectType,HasUniqueRoleAssignments" --output json --query "[?HasUniqueRoleAssignments == \`true\`]"
                    if ($LASTEXITCODE -eq 0) {
                        $items = @($itemsJson | ConvertFrom-Json)
                        
                        if ($items.Count -gt 0) {
                            Write-Host "    Found $($items.Count) items with unique permissions in list: $($list.Title)" -ForegroundColor Yellow
                            
                            foreach ($item in $items) {
                                try {
                                    $fileUrl = $item.FileRef
                                    $fileName = $item.FileLeafRef
                                    $itemType = if ($item.FileSystemObjectType -eq 1) { "Folder" } else { "File" }
                                    
                                    # Get item RoleAssignments via REST API
                                    $itemRoleAssignmentsJson = m365 request --url "$SiteUrl/_api/web/lists/getbytitle('$($list.Title)')/items($($item.ID))/roleassignments?\`$expand=Member,RoleDefinitionBindings" --output json
                                    if ($LASTEXITCODE -eq 0) {
                                        $itemRoleAssignmentsData = $itemRoleAssignmentsJson | ConvertFrom-Json
                                        $itemRoleAssignments = @($itemRoleAssignmentsData.value)
                                        
                                        foreach ($roleAssignment in $itemRoleAssignments) {
                                            $member = $roleAssignment.Member
                                            $roles = ($roleAssignment.RoleDefinitionBindings | ForEach-Object { $_.Name }) -join ","
                                            
                                            if ($ExcludeLimitedAccess -and $roles -eq "Limited Access") {
                                                continue
                                            }
                                            
                                            $memberType = switch ($member.PrincipalType) {
                                                1 { "User" }
                                                4 { "SecurityGroup" }
                                                8 { "SharePointGroup" }
                                                default { "Unknown" }
                                            }
                                            
                                            [void]$script:PermissionsReport.Add([PSCustomObject]@{
                                                SiteUrl = $SiteUrl
                                                SiteTitle = $siteTitle
                                                Type = $itemType
                                                SensitivityLabel = $sensitivityLabel
                                                RelativeUrl = $fileUrl
                                                ListTitle = $list.Title
                                                MemberType = $memberType
                                                MemberName = $member.Title
                                                MemberLoginName = $member.LoginName
                                                ParentGroup = if ($memberType -eq "User") { "NA" } else { $member.Title }
                                                Roles = $roles
                                            })
                                        }
                                    }
                                    
                                    # Get sharing links for files
                                    if ($itemType -eq "File") {
                                        try {
                                            $sharingLinksJson = m365 spo file sharinglink list --webUrl $SiteUrl --fileUrl $fileUrl --output json
                                            if ($LASTEXITCODE -eq 0) {
                                                $sharingLinks = @($sharingLinksJson | ConvertFrom-Json)
                                                
                                                foreach ($link in $sharingLinks) {
                                                    [void]$script:PermissionsReport.Add([PSCustomObject]@{
                                                        SiteUrl = $SiteUrl
                                                        SiteTitle = $siteTitle
                                                        Type = $itemType
                                                        SensitivityLabel = $sensitivityLabel
                                                        RelativeUrl = $fileUrl
                                                        ListTitle = $list.Title
                                                        MemberType = "Sharing Link"
                                                        MemberName = $link.scope
                                                        MemberLoginName = $link.shareLink.webUrl
                                                        ParentGroup = ""
                                                        Roles = $link.shareLink.type
                                                    })
                                                    $script:Summary.SharingLinksFound++
                                                }
                                            }
                                        }
                                        catch {
                                            Write-Verbose "      No sharing links found for file: $fileName"
                                        }
                                    }
                                    
                                    # Get sharing links for folders
                                    if ($itemType -eq "Folder") {
                                        try {
                                            $sharingLinksJson = m365 spo folder sharinglink list --webUrl $SiteUrl --folderUrl $fileUrl --output json
                                            if ($LASTEXITCODE -eq 0) {
                                                $sharingLinks = @($sharingLinksJson | ConvertFrom-Json)
                                                
                                                foreach ($link in $sharingLinks) {
                                                    [void]$script:PermissionsReport.Add([PSCustomObject]@{
                                                        SiteUrl = $SiteUrl
                                                        SiteTitle = $siteTitle
                                                        Type = $itemType
                                                        SensitivityLabel = $sensitivityLabel
                                                        RelativeUrl = $fileUrl
                                                        ListTitle = $list.Title
                                                        MemberType = "Sharing Link"
                                                        MemberName = $link.scope
                                                        MemberLoginName = $link.shareLink.webUrl
                                                        ParentGroup = ""
                                                        Roles = $link.shareLink.type
                                                    })
                                                    $script:Summary.SharingLinksFound++
                                                }
                                            }
                                        }
                                        catch {
                                            Write-Verbose "      No sharing links found for folder: $fileName"
                                        }
                                    }
                                    
                                    $script:Summary.ItemsAudited++
                                }
                                catch {
                                    Write-Warning "      Error processing item ID $($item.ID): $_"
                                    $script:Summary.Failures++
                                    continue
                                }
                            }
                        }
                        else {
                            Write-Verbose "    No items with unique permissions found"
                        }
                    }
                }
            }
            catch {
                Write-Warning "  Error processing list '$($list.Title)': $_"
                $script:Summary.Failures++
                continue
            }
        }
        
        Write-Progress -Activity "Auditing Lists" -Completed
    }
    catch {
        Write-Host "\nFailed to complete permission audit: $_" -ForegroundColor Red
        throw
    }
}

end {
    Write-Host "\nExporting permission audit report..." -ForegroundColor Yellow
    
    if ($script:PermissionsReport.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "PermissionAudit_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
        $script:PermissionsReport | Select-Object SiteUrl, SiteTitle, Type, SensitivityLabel, RelativeUrl, ListTitle, MemberType, MemberName, MemberLoginName, ParentGroup, Roles | Export-Csv -Path $csvPath -NoTypeInformation
        
        Write-Host "  Permission audit report exported to: " -NoNewline -ForegroundColor Green
        Write-Host $csvPath -ForegroundColor Cyan
    }
    else {
        Write-Host "  No permissions found to export" -ForegroundColor Yellow
    }
    
    Write-Host "\n=== Permission Audit Summary ===" -ForegroundColor Cyan
    Write-Host "Sites Audited: " -NoNewline
    Write-Host $script:Summary.SitesAudited -ForegroundColor Green
    Write-Host "Lists Audited: " -NoNewline
    Write-Host $script:Summary.ListsAudited -ForegroundColor Green
    Write-Host "Items Audited: " -NoNewline
    Write-Host $script:Summary.ItemsAudited -ForegroundColor Green
    Write-Host "Sharing Links Found: " -NoNewline
    if ($script:Summary.SharingLinksFound -gt 0) {
        Write-Host $script:Summary.SharingLinksFound -ForegroundColor Yellow
    } else {
        Write-Host $script:Summary.SharingLinksFound -ForegroundColor Green
    }
    Write-Host "Failures: " -NoNewline
    if ($script:Summary.Failures -gt 0) {
        Write-Host $script:Summary.Failures -ForegroundColor Red
    } else {
        Write-Host $script:Summary.Failures -ForegroundColor Green
    }
    Write-Host "Log File: " -NoNewline
    Write-Host $logPath -ForegroundColor Cyan
    Write-Host "================================\n" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Example 1: Audit site-level and list-level permissions
# .\Get-PermissionAudit.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing"

# Example 2: Audit with item-level permissions (comprehensive)
# .\Get-PermissionAudit.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -IncludeListItems

# Example 3: Audit with custom output path and exclude Limited Access
# .\Get-PermissionAudit.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/HR" -OutputPath "C:\\Reports" -ExcludeLimitedAccess

# Example 4: Comprehensive audit with verbose output for troubleshooting
# .\Get-PermissionAudit.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -IncludeListItems -ExcludeLimitedAccess -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***



[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [PowerShell Script to Query Unique Permissions in SharePoint](https://reshmeeauckloo.com/posts/powershell-query-unique-permissions-sharepoint/)

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Reshmee Auckloo](https://github.com/reshmee011) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-permission-audit" aria-hidden="true" />



# Deletes company-wide and anonymous sharing links

## Summary

Sharing links can lead to oversharing, especially when default site sharing settings haven’t been updated to ‘People with existing access’. If default sharing options are not updated in the tenant or site, end users can easily create a company wide or anonymous sharing links with a single click. End users can make a conscious decision to create sharing links with specific "people you choose" which limits the audience having access to the data. This script can help delete those company-wide and anonymous sharing links at folder, file, and item levels. This approach can help mitigate oversharing issues during the M365 Copilot rollout.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have at least site owner role to the SharePoint Online site.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "SharePoint site URL to scan for sharing links")]
    [ValidateNotNullOrEmpty()]
    [string]$SiteUrl,
    
    [Parameter(HelpMessage = "Scope of links to remove: Both, Anonymous, or Organization")]
    [ValidateSet('Both', 'Anonymous', 'Organization')]
    [string]$Scope = 'Both',
    
    [Parameter(HelpMessage = "Export results to CSV file")]
    [switch]$ExportToCsv,
    
    [Parameter(HelpMessage = "CSV output file path")]
    [string]$OutputPath = "SharingLinksReport-$(Get-Date -Format 'yyyyMMdd-HHmmss').csv"
)

begin {
    $transcript = "SharingLinksRemoval-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
    Start-Transcript -Path $transcript
    
    Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Starting sharing link removal process..." -ForegroundColor Cyan
    
    Write-Verbose "Ensuring Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365. Please run 'm365 login' first."
    }
    
    $script:Summary = @{
        ListsProcessed = 0
        ItemsScanned = 0
        LinksFound = 0
        LinksRemoved = 0
        Failures = 0
    }
    
    $script:Report = [System.Collections.Generic.List[PSObject]]::new()
    
    Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Configuration:" -ForegroundColor Yellow
    Write-Host "  Site URL: $SiteUrl" -ForegroundColor Gray
    Write-Host "  Scope: $Scope" -ForegroundColor Gray
    if ($WhatIfPreference) {
        Write-Host "  WhatIf: Enabled (simulation mode)" -ForegroundColor Magenta
    }
}

process {
    try {
        Write-Verbose "Retrieving non-hidden lists and libraries from site..."
        $listsJson = m365 spo list list --webUrl $SiteUrl --filter "Hidden eq false" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve lists: $listsJson"
        }
        $lists = @($listsJson | ConvertFrom-Json)
        
        Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Found $($lists.Count) list(s)/library(ies) to scan" -ForegroundColor Green
        
        foreach ($list in $lists) {
            $script:Summary.ListsProcessed++
            Write-Verbose "Processing list: $($list.Title) (ID: $($list.Id))"
            
            try {
                $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listId $list.Id --fields "FileRef,FSObjType,HasUniqueRoleAssignments" --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve items from list '$($list.Title)': $itemsJson"
                    continue
                }
                $items = @($itemsJson | ConvertFrom-Json)
                
                $items = $items | Where-Object { $_.HasUniqueRoleAssignments -eq $true }
                
                if ($items.Count -eq 0) {
                    Write-Verbose "  List '$($list.Title)' has no items with unique permissions, skipping..."
                    continue
                }
                
                Write-Host "  [$(Get-Date -Format 'HH:mm:ss')] Scanning $($items.Count) item(s) in '$($list.Title)'..." -ForegroundColor Gray
                
                foreach ($item in $items) {
                    $script:Summary.ItemsScanned++
                    $itemPath = $item.FileRef
                    $isFolder = $item.FSObjType -eq 1
                    $itemType = if ($isFolder) { 'Folder' } else { 'File' }
                    
                    Write-Verbose "    Checking $itemType`: $itemPath"
                    
                    try {
                        $linksJson = if ($isFolder) {
                            m365 spo folder sharinglink list --webUrl $SiteUrl --folderUrl $itemPath --output json 2>&1
                        } else {
                            m365 spo file sharinglink list --webUrl $SiteUrl --fileUrl $itemPath --output json 2>&1
                        }
                        
                        if ($LASTEXITCODE -ne 0) {
                            Write-Verbose "      No sharing links or error retrieving links for $itemPath"
                            continue
                        }
                        
                        $allLinks = @($linksJson | ConvertFrom-Json)
                        
                        $targetLinks = $allLinks | Where-Object {
                            $linkScope = $_.link.scope
                            if ($Scope -eq 'Both') {
                                $linkScope -eq 'anonymous' -or $linkScope -eq 'organization'
                            } elseif ($Scope -eq 'Anonymous') {
                                $linkScope -eq 'anonymous'
                            } elseif ($Scope -eq 'Organization') {
                                $linkScope -eq 'organization'
                            }
                        }
                        
                        if ($targetLinks.Count -eq 0) {
                            continue
                        }
                        
                        $script:Summary.LinksFound += $targetLinks.Count
                        Write-Host "    Found $($targetLinks.Count) $Scope link(s) on $itemType`: $(Split-Path $itemPath -Leaf)" -ForegroundColor Yellow
                        
                        foreach ($link in $targetLinks) {
                            $reportEntry = [PSCustomObject]@{
                                SiteUrl = $SiteUrl
                                ListTitle = $list.Title
                                ItemPath = $itemPath
                                ItemName = (Split-Path $itemPath -Leaf)
                                ItemType = $itemType
                                LinkId = $link.id
                                LinkScope = $link.link.scope
                                LinkType = $link.link.type
                                Users = (($link.grantedToIdentitiesV2.user.email | Where-Object { $_ }) -join '|')
                                Roles = (($link.roles | Where-Object { $_ }) -join '|')
                                ShareLinkUrl = $link.link.webUrl
                                Expiration = $link.expirationDateTime
                                RequiresPassword = $link.hasPassword
                                PreventsDownload = $link.link.preventsDownload
                                Status = 'Found'
                                Error = ''
                            }
                            
                            $scopeArg = switch ($link.link.scope) {
                                'anonymous' { 'anonymous' }
                                'organization' { 'organization' }
                                default { '' }
                            }
                            
                            if ($PSCmdlet.ShouldProcess("$itemType '$itemPath'", "Remove $($link.link.scope) sharing link")) {
                                try {
                                    if ($isFolder) {
                                        m365 spo folder sharinglink clear --webUrl $SiteUrl --folderUrl $itemPath --scope $scopeArg --force 2>&1 | Out-Null
                                    } else {
                                        m365 spo file sharinglink clear --webUrl $SiteUrl --fileUrl $itemPath --scope $scopeArg --force 2>&1 | Out-Null
                                    }
                                    
                                    if ($LASTEXITCODE -eq 0) {
                                        $script:Summary.LinksRemoved++
                                        $reportEntry.Status = 'Removed'
                                        Write-Host "      Removed $($link.link.scope) link from $itemType" -ForegroundColor Green
                                    } else {
                                        $script:Summary.Failures++
                                        $reportEntry.Status = 'Failed'
                                        $reportEntry.Error = 'Removal failed'
                                        Write-Warning "Failed to remove link from $itemPath"
                                    }
                                } catch {
                                    $script:Summary.Failures++
                                    $reportEntry.Status = 'Failed'
                                    $reportEntry.Error = $_.Exception.Message
                                    Write-Warning "Error removing link from $itemPath: $($_.Exception.Message)"
                                }
                            } else {
                                $reportEntry.Status = 'WhatIf'
                            }
                            
                            $script:Report += $reportEntry
                        }
                    } catch {
                        Write-Warning "Error processing $itemType $itemPath: $($_.Exception.Message)"
                    }
                }
            } catch {
                Write-Warning "Failed to process list '$($list.Title)': $($_.Exception.Message)"
            }
        }
    } catch {
        Write-Error "Critical error during processing: $($_.Exception.Message)"
        throw
    }
}

end {
    Write-Host ""
    Write-Host "=== Sharing Link Removal Summary ===" -ForegroundColor Cyan
    Write-Host "  Lists processed: $($script:Summary.ListsProcessed)" -ForegroundColor White
    Write-Host "  Items scanned: $($script:Summary.ItemsScanned)" -ForegroundColor White
    Write-Host "  (Only items with unique permissions - filtering saved ~90% of API calls)" -ForegroundColor Gray
    Write-Host "  Links found: $($script:Summary.LinksFound)" -ForegroundColor White
    
    if ($WhatIfPreference) {
        Write-Host "  Links would be removed: $($script:Summary.LinksFound)" -ForegroundColor Magenta
        Write-Host "  (WhatIf mode - no actual changes made)" -ForegroundColor Magenta
    } else {
        Write-Host "  Links removed: $($script:Summary.LinksRemoved)" -ForegroundColor Green
        if ($script:Summary.Failures -gt 0) {
            Write-Host "  Failures: $($script:Summary.Failures)" -ForegroundColor Red
        } else {
            Write-Host "  Failures: 0" -ForegroundColor Green
        }
    }
    
    if ($ExportToCsv -and $script:Report.Count -gt 0) {
        $script:Report | Export-Csv -Path $OutputPath -NoTypeInformation
        Write-Host ""
        Write-Host "Report exported to: $OutputPath" -ForegroundColor Green
    }
    
    Write-Host ""
    Write-Host "Transcript saved to: $transcript" -ForegroundColor Gray
    Stop-Transcript
}

# Usage Examples:

# Example 1: Preview what would be removed (WhatIf mode)
# .\Remove-SharingLinks.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -Scope Anonymous -WhatIf

# Example 2: Remove anonymous links with verbose output
# .\Remove-SharingLinks.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -Scope Anonymous -Verbose

# Example 3: Remove organization links and export to CSV
# .\Remove-SharingLinks.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -Scope Organization -ExportToCsv

# Example 4: Remove both types with verbose output and CSV export
# .\Remove-SharingLinks.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -Scope Both -Verbose -ExportToCsv
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
param(
    [Parameter(Mandatory)]
    [ValidateSet('Yes','No')]
    [string]$ExcludeDirectSharingLink,#anonymous and company wide links are included by default and this provides an option to include the "People You choose" links in the deletio process.
    [Parameter(Mandatory)]
    [ValidateSet('Yes','No')]
    [string]$DeleteSharingink,
    [Parameter(Mandatory)]
    [string]$SiteUrl
)
 
#Parameters
$dateTime = (Get-Date).toString("dd-MM-yyyy-hh-ss")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "SharedLinksDeletion-" + $dateTime + ".csv"
$ReportOutput = $directorypath + "\Logs\"+ $fileName 
 
$global:Results = @();

$siteUrl = if ($siteUrl[-1] -ne '/') { $siteUrl + '/' } else { $siteUrl }
function getSharingLink($_object,$_type,$_siteUrl,$_listUrl)
{
    $relativeUrl = $_object.FileRef
    $SharingLinks = if ($_type -eq 0 ) {
        Get-PnPFileSharingLink -Identity $relativeUrl
    } elseif ($_type -eq 1) {
        Get-PnPFolderSharingLink -Folder $relativeUrl
    }
   
    ForEach($ShareLink in $SharingLinks)
    {
        if(($ExcludeDirectSharingLink -eq 'Yes' -and $ShareLink.Link.Scope -ne 'Users') -or $ExcludeDirectSharingLink -eq 'No')
        {
            $result = New-Object PSObject -property $([ordered]@{
                SiteUrl = $_SiteURL
                listUrl = $_listUrl
                Name =  $_object.FileLeafRef      
                RelativeURL = $_object.FileRef
                ObjectType = $_Type -eq 1 ? 'Folder':'File'
                ShareId = $ShareLink.Id
                RoleList = $ShareLink.Roles -join "|"
                Users = $ShareLink.GrantedToIdentitiesV2.User.Email -join "|"
                ShareLinkUrl  = $ShareLink.Link.WebUrl
                ShareLinkType  = $ShareLink.Link.Type
                ShareLinkScope  = $ShareLink.Link.Scope
                Expiration = $ShareLink.ExpirationDateTime
                BlocksDownload = $ShareLink.Link.PreventsDowload
                RequiresPassword = $ShareLink.HasPassword
            })
       
            $global:Results +=$result;
 
        if($DeleteSharingink -eq 'Yes'){
            if ($_type -eq 0 ) {
                Remove-PnPFileSharingLink -FileUrl $relativeUrl -Identity $ShareLink.Id -Force
            } elseif ($_type -eq 1) {
                Remove-PnPFolderSharingLink -Folder $relativeUrl -Identity $ShareLink.Id -Force
            } 
      }
    }    
 }
}
#Exclude certain libraries/lists
$ExcludedLists = @("Access Requests", "App Packages", "appdata", "appfiles","Apps for SharePoint" ,"Apps in Testing", "Cache Profiles", "Composed Looks", "Content and Structure Reports", "Content type publishing error log", "Converted Forms",
    "Device Channels", "Form Templates", "fpdatasources", "Get started with Apps for Office and SharePoint", "List Template Gallery", "Long Running Operation Status", "Maintenance Log Library", "Images", "site collection images"
    , "Master Docs", "Master Page Gallery", "MicroFeed", "NintexFormXml", "Quick Deploy Items", "Relationships List", "Reusable Content", "Reporting Metadata", "Reporting Templates", "Search Config List", "Site Assets", "Preservation Hold Library",
    "Site Pages", "Solution Gallery", "Style Library", "Suggested Content Browser Locations", "Theme Gallery", "TaxonomyHiddenList", "User Information List", "Web Part Gallery", "wfpub", "wfsvc", "Workflow History", "Workflow Tasks", "Pages")
 
Connect-PnPOnline -Url $siteUrl -Interactive
 
Write-Host "Processing site $siteUrl"  -Foregroundcolor "Red";
$ll = Get-PnPList -Includes BaseType, Hidden, Title,HasUniqueRoleAssignments,RootFolder | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists } #$_.BaseType -eq "DocumentLibrary"
  Write-Host "Number of lists $($ll.Count)";
 
  foreach($list in $ll)
  {
    $listUrl = $list.RootFolder.ServerRelativeUrl;    
 
    $selectFields = "ID,HasUniqueRoleAssignments,FileRef,FileLeafRef,FileSystemObjectType"
    
    $Url = $siteUrl + '_api/web/lists/getbytitle(''' + $($list.Title) + ''')/items?$select=' + $($selectFields)
    $nextLink = $Url
    $ListItems = @()
    while($nextLink){  
        $response = invoke-pnpsprestmethod -Url $nextLink -Method Get
    
        $ListItems += $response.value | where-object{$_.HasUniqueRoleAssignments -eq $true}
        if($response.'odata.nextlink'){
            $nextLink = $response.'odata.nextlink' -replace "$siteUrl/_api/",""
        }    else{
            $nextLink = $null
        }    
    }
    ForEach($item in $ListItems)
    {
        if($list.BaseType -eq "DocumentLibrary")
        {
            $type= $item.FileSystemObjectType;
        }
        
        getSharingLink $item $type $siteUrl $listUrl;
    }
}
 
$global:Results | Export-CSV $ReportOutput -NoTypeInformation
Write-host -f Green "Sharing Links Report Generated Successfully! to $ReportOutput"
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Deletion of company-wide and anonymous sharing links with PowerShell](https://reshmeeauckloo.com/posts/powershell-delete-nondirectlink//)

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Reshmee Auckloo](https://github.com/reshmee011) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-delete-companywide-anonymous-sharinglink" aria-hidden="true" />

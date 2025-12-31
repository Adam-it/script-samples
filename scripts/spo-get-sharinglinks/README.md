

# Get sharing links within the tenant

## Summary

Effective oversight of sharing links is paramount to ensuring data security, compliance, and optimal collaboration experiences.

For Copilot for M365 implementations, ensuring there is no oversharing is a critical aspect of safeguarding sensitive information and maintaining regulatory compliance. By integrating the sharing link audit process into deployment strategies, administrators can preemptively address security vulnerabilities and uphold the integrity of M365 environments.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have SharePoint Online tenant administrator access.


# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Tenant admin center URL (e.g., https://contoso-admin.sharepoint.com)")]
    [string]$TenantAdminUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Path for the CSV export file")]
    [string]$OutputPath,

    [Parameter(Mandatory = $false, HelpMessage = "Export results to CSV instead of displaying in console")]
    [switch]$ExportToCsv
)

begin {
    # Initialize summary tracking
    $script:Summary = [PSCustomObject]@{
        SitesScanned = 0
        ListsScanned = 0
        ItemsScanned = 0
        LinksFound = 0
    }

    $script:Results = [System.Collections.Generic.List[PSObject]]::new()

    # Start transcript for logging
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = "Get-SharingLinks-$timestamp.log"
    Start-Transcript -Path $transcriptPath -Append

    Write-Host "Starting sharing link scan..." -ForegroundColor Cyan
    Write-Host "Tenant Admin URL: $TenantAdminUrl" -ForegroundColor White

    # Ensure user is logged in to CLI for Microsoft 365
    Write-Verbose "Ensuring CLI for Microsoft 365 connection..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' first."
    }
    Write-Verbose "Successfully authenticated to CLI for Microsoft 365"

    # Get all M365 sites (exclude redirects and deleted sites)
    Write-Host "Retrieving SharePoint sites..." -ForegroundColor Yellow
    $sitesJson = m365 spo site list --filter "Template ne 'RedirectSite#0'" --output json
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve SharePoint sites. Error: $sitesJson"
    }
    $sites = @($sitesJson | ConvertFrom-Json)
    # Filter to only /sites/ URLs to match PnP PowerShell behavior
    $sites = $sites | Where-Object { $_.Url -like '*/sites/*' }
    Write-Host "Found $($sites.Count) site(s) to scan" -ForegroundColor Green

    # Excluded system lists per AGENTS.md guidance
    $script:ExcludedLists = @(
        "Access Requests", "App Packages", "appdata", "appfiles", "Apps in Testing",
        "Cache Profiles", "Composed Looks", "Content and Structure Reports",
        "Content type publishing error log", "Converted Forms", "Device Channels",
        "Form Templates", "fpdatasources", "Get started with Apps for Office and SharePoint",
        "List Template Gallery", "Long Running Operation Status", "Maintenance Log Library",
        "Images", "site collection images", "Master Docs", "Master Page Gallery",
        "MicroFeed", "NintexFormXml", "Quick Deploy Items", "Relationships List",
        "Reusable Content", "Reporting Metadata", "Reporting Templates", "Search Config List",
        "Site Assets", "Preservation Hold Library", "Site Pages", "Solution Gallery",
        "Style Library", "Suggested Content Browser Locations", "Theme Gallery",
        "TaxonomyHiddenList", "User Information List", "Web Part Gallery", "wfpub",
        "wfsvc", "Workflow History", "Workflow Tasks", "Pages"
    )
}

process {
    foreach ($site in $sites) {
        $siteUrl = $site.Url
        $script:Summary.SitesScanned++

        Write-Host "`nProcessing site: $siteUrl" -ForegroundColor Cyan
        Write-Verbose "Site $($script:Summary.SitesScanned) of $($sites.Count)"

        try {
            # Get all non-hidden lists/libraries
            Write-Verbose "Retrieving lists for $siteUrl..."
            $listsJson = m365 spo list list --webUrl $siteUrl --filter "Hidden eq false" --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve lists for $siteUrl. Skipping site. Error: $listsJson"
                continue
            }
            $lists = @($listsJson | ConvertFrom-Json) | Where-Object { $_.Title -notin $script:ExcludedLists }
            Write-Verbose "Found $($lists.Count) list(s) to scan"

            foreach ($list in $lists) {
                $script:Summary.ListsScanned++
                $listTitle = $list.Title
                $listId = $list.Id

                Write-Verbose "  Scanning list: $listTitle"

                try {
                    # Get list items with HasUniqueRoleAssignments filter (performance optimization per AGENTS.md)
                    $itemsJson = m365 spo listitem list --webUrl $siteUrl --listId $listId --fields "FileRef,FSObjType,HasUniqueRoleAssignments" --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "    Failed to retrieve items for list '$listTitle'. Skipping. Error: $itemsJson"
                        continue
                    }
                    $items = @($itemsJson | ConvertFrom-Json) | Where-Object { $_.HasUniqueRoleAssignments -eq $true }
                    $script:Summary.ItemsScanned += $items.Count
                    Write-Verbose "    Found $($items.Count) item(s) with unique permissions"

                    foreach ($item in $items) {
                        $itemUrl = $item.FileRef
                        $isFolder = $item.FSObjType -eq 1
                        $objectType = if ($isFolder) { "Folder" } else { "File" }

                        Write-Verbose "      Checking $objectType: $itemUrl"

                        try {
                            # Get sharing links based on object type
                            if ($isFolder) {
                                $linksJson = m365 spo folder sharinglink list --webUrl $siteUrl --folderUrl $itemUrl --output json 2>&1
                            } else {
                                $linksJson = m365 spo file sharinglink list --webUrl $siteUrl --fileUrl $itemUrl --output json 2>&1
                            }

                            if ($LASTEXITCODE -ne 0) {
                                Write-Verbose "        No sharing links or error for $itemUrl"
                                continue
                            }

                            $links = @($linksJson | ConvertFrom-Json)
                            if ($links.Count -eq 0) {
                                Write-Verbose "        No sharing links found"
                                continue
                            }

                            Write-Verbose "        Found $($links.Count) sharing link(s)"

                            foreach ($link in $links) {
                                $script:Summary.LinksFound++

                                # Extract users from grantedToIdentitiesV2
                                $users = ($link.grantedToIdentitiesV2.user.email | Where-Object { $_ }) -join '|'
                                
                                # Extract item name from URL
                                $itemName = Split-Path -Leaf $itemUrl

                                $result = [PSCustomObject]@{
                                    SiteUrl = $siteUrl
                                    ListTitle = $listTitle
                                    ItemName = $itemName
                                    RelativeURL = $itemUrl
                                    ObjectType = $objectType
                                    ShareId = $link.id
                                    Roles = ($link.roles | Where-Object { $_ }) -join '|'
                                    Users = $users
                                    ShareLinkUrl = $link.link.webUrl
                                    ShareLinkType = $link.link.type
                                    ShareLinkScope = $link.link.scope
                                    Expiration = $link.expirationDateTime
                                    PreventsDownload = $link.link.preventsDownload
                                    RequiresPassword = $link.hasPassword
                                }

                                $script:Results.Add($result)
                            }
                        } catch {
                            Write-Warning "      Error processing $objectType '$itemUrl': $_"
                            continue
                        }
                    }
                } catch {
                    Write-Warning "  Error processing list '$listTitle': $_"
                    continue
                }
            }
        } catch {
            Write-Warning "Error processing site '$siteUrl': $_"
            continue
        }
    }
}

end {
    # Display summary
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Sharing Links Scan Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Sites scanned:                 $($script:Summary.SitesScanned)" -ForegroundColor White
    Write-Host "Lists scanned:                 $($script:Summary.ListsScanned)" -ForegroundColor White
    Write-Host "Items scanned (unique perms):  $($script:Summary.ItemsScanned)" -ForegroundColor White
    Write-Host "Sharing links found:           $($script:Summary.LinksFound)" -ForegroundColor Green
    Write-Host "========================================" -ForegroundColor Cyan

    # Export or display results
    if ($script:Results.Count -gt 0) {
        if ($ExportToCsv) {
            if (-not $OutputPath) {
                $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
                $OutputPath = "SharingLinks-$timestamp.csv"
            }

            $script:Results | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
            Write-Host "Results exported to: $OutputPath" -ForegroundColor Green
        } else {
            Write-Host "`nSharing Links (first 50 results):" -ForegroundColor Yellow
            $script:Results | Select-Object -First 50 | Format-Table -AutoSize
            if ($script:Results.Count -gt 50) {
                Write-Host "... and $($script:Results.Count - 50) more. Use -ExportToCsv to see all results." -ForegroundColor Gray
            }
        }
    } else {
        Write-Host "No sharing links found in the scanned sites." -ForegroundColor Yellow
    }

    Stop-Transcript
    Write-Host "`nTranscript saved to: $transcriptPath" -ForegroundColor Gray
}

# Usage examples:
# .\\Get-SharingLinks.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -Verbose
# .\\Get-SharingLinks.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -ExportToCsv
# .\\Get-SharingLinks.ps1 -TenantAdminUrl "https://contoso-admin.sharepoint.com" -ExportToCsv -OutputPath "C:\\Reports\\SharingLinks.csv"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
#Parameters
$tenantUrl = Read-Host -Prompt "Enter tenant collection URL";
$dateTime = (Get-Date).toString("dd-MM-yyyy-hh-ss")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "SharedLinks-" + $dateTime + ".csv"
$ReportOutput = $directorypath + "\Logs\"+ $fileName

#Connect to PnP Online
Connect-PnPOnline -Url $tenantUrl -Interactive

$global:Results = @();

function getSharingLink($_object,$_type,$_siteUrl,$_listUrl)
{
    $relativeUrl = $_object.FieldValues["FileRef"]
    $SharingLinks = if ($_type -eq "File" -or $_type -eq "Item") {
        Get-PnPFileSharingLink -Identity $relativeUrl
    } elseif ($_type -eq "Folder") {
        Get-PnPFolderSharingLink -Folder $relativeUrl
    }
    
    ForEach($ShareLink in $SharingLinks)
    {
        $result = New-Object PSObject -property $([ordered]@{
            SiteUrl = $_SiteURL
            listUrl = $_listUrl
            Name = $_type -eq 'Item' ? $_object.FieldValues["Title"] : $_object.FieldValues["FileLeafRef"]          
            RelativeURL = $_object.FieldValues["FileRef"] 
            ObjectType = $_Type
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
    }     
}

#Exclude certain libraries
$ExcludedLists = @("Access Requests", "App Packages", "appdata", "appfiles", "Apps in Testing", "Cache Profiles", "Composed Looks", "Content and Structure Reports", "Content type publishing error log", "Converted Forms",
    "Device Channels", "Form Templates", "fpdatasources", "Get started with Apps for Office and SharePoint", "List Template Gallery", "Long Running Operation Status", "Maintenance Log Library", "Images", "site collection images"
    , "Master Docs", "Master Page Gallery", "MicroFeed", "NintexFormXml", "Quick Deploy Items", "Relationships List", "Reusable Content", "Reporting Metadata", "Reporting Templates", "Search Config List", "Site Assets", "Preservation Hold Library",
    "Site Pages", "Solution Gallery", "Style Library", "Suggested Content Browser Locations", "Theme Gallery", "TaxonomyHiddenList", "User Information List", "Web Part Gallery", "wfpub", "wfsvc", "Workflow History", "Workflow Tasks", "Pages")

$m365Sites = Get-PnPTenantSite| Where-Object { ( $_.Url -like '*/sites/*') -and $_.Template -ne 'RedirectSite#0' } 
$m365Sites | ForEach-Object {
$siteUrl = $_.Url;     
Connect-PnPOnline -Url $siteUrl -Interactive

Write-Host "Processing site $siteUrl"  -Foregroundcolor "Red"; 

#getSharingLink $ctx $web "site" $siteUrl "";
$ll = Get-PnPList -Includes BaseType, Hidden, Title,HasUniqueRoleAssignments,RootFolder | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists } #$_.BaseType -eq "DocumentLibrary" 
  Write-Host "Number of lists $($ll.Count)";

  foreach($list in $ll)
  {
    $listUrl = $list.RootFolder.ServerRelativeUrl;       

    #Get all list items in batches
    $ListItems = Get-PnPListItem -List $list -PageSize 2000 

        ForEach($item in $ListItems)
        {
            #Check if the Item has unique permissions
            $HasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
            If($HasUniquePermissions)
            {       
                #Get Shared Links
                if($list.BaseType -eq "DocumentLibrary")
                {
                    $type= $item.FileSystemObjectType;
                }
                else
                {
                    $type= "Item";
                }
                getSharingLink $item $type $siteUrl $listUrl;
            }
        }
    }
 }
 
 $global:Results | Export-CSV $ReportOutput -NoTypeInformation
  #Export-CSV $ReportOutput -NoTypeInformation
Write-host -f Green "Sharing Links Report Generated Successfully!"
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Oversight of Sharing Links in SharePoint sites using PowerShell](https://reshmeeauckloo.com/posts/powershell-get-sharing-links-sharepoint/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-sharinglinks" aria-hidden="true" />

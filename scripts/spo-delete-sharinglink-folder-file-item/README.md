

# Deletes sharing links for folder, file and item

## Summary

Sharing links can lead to oversharing, especially when default site sharing settings haven't been updated to 'People with existing access.' To address this, consider using a utility script that deletes sharing links at the folder, file, and item levels. This approach can help mitigate oversharing issues during the Copilot for M365 rollout. This sample includes both PnP PowerShell and CLI for Microsoft 365 implementations.

![Example Screenshot](assets/preview.png)

### Prerequisites

- The user account that runs the script must have access to the SharePoint Online site.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "SharePoint site URL")]
    [string]$SiteUrl,
    
    [Parameter(HelpMessage = "Scope of sharing links to delete (anonymous, users, organization). All scopes if not specified")]
    [ValidateSet("anonymous", "users", "organization")]
    [string]$Scope,
    
    [Parameter(HelpMessage = "Export deleted sharing links to CSV")]
    [switch]$ExportToCsv,
    
    [Parameter(HelpMessage = "Path for CSV export")]
    [string]$OutputPath = "SharingLinkDeletionReport-$(Get-Date -Format 'dd-MM-yyyy-HH-mm').csv"
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' manually."
    }

    $ExcludedLists = @(
        "Form Templates", "Preservation Hold Library", "Site Assets", 
        "Images", "Pages", "Settings", "Videos", "Style Library", 
        "AppPages", "Apps for SharePoint", "Apps for Office"
    )

    $script:Summary = @{
        LibrariesProcessed = 0
        FilesProcessed = 0
        FoldersProcessed = 0
        SharingLinksFound = 0
        SharingLinksDeleted = 0
        Failed = 0
    }

    $script:Results = [System.Collections.ArrayList]::new()
}

process {
    Write-Host "Processing site: $SiteUrl" -ForegroundColor Cyan
    
    Write-Verbose "Retrieving document libraries..."
    $listsJson = m365 spo list list --webUrl $SiteUrl --filter "Hidden eq false and BaseTemplate eq 101" --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve lists from site. CLI: $listsJson"
    }
    
    $lists = @($listsJson | ConvertFrom-Json) | Where-Object { $_.Title -notin $ExcludedLists }
    Write-Host "Found $($lists.Count) document libraries to process" -ForegroundColor Cyan
    
    if ($lists.Count -eq 0) {
        Write-Host "No libraries to process" -ForegroundColor Yellow
        return
    }
    
    foreach ($list in $lists) {
        $script:Summary.LibrariesProcessed++
        Write-Host "`nProcessing library: $($list.Title)" -ForegroundColor Cyan
        
        try {
            Write-Verbose "Retrieving items with unique permissions..."
            $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listId $list.Id --fields "FileRef,FileSystemObjectType,HasUniqueRoleAssignments" --query "[?HasUniqueRoleAssignments == \`$true\`]" --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve items from library '$($list.Title)'. CLI: $itemsJson"
                $script:Summary.Failed++
                continue
            }
            
            $items = @($itemsJson | ConvertFrom-Json)
            
            if ($items.Count -eq 0) {
                Write-Verbose "  No items with unique permissions found"
                continue
            }
            
            Write-Host "  Found $($items.Count) items with unique permissions" -ForegroundColor Gray
            
            foreach ($item in $items) {
                $itemPath = $item.FileRef
                $itemType = if ($item.FileSystemObjectType -eq 0) { "File" } else { "Folder" }
                
                try {
                    if ($itemType -eq "File") {
                        $script:Summary.FilesProcessed++
                        
                        Write-Verbose "    Checking file: $itemPath"
                        $queryFilter = if ($Scope) { " --query `"[?link.scope == '$Scope']`"" } else { "" }
                        $sharingLinksJson = m365 spo file sharinglink list --webUrl $SiteUrl --fileUrl $itemPath$queryFilter --output json 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "    Failed to list sharing links for file '$itemPath'. CLI: $sharingLinksJson"
                            $script:Summary.Failed++
                            continue
                        }
                        
                        $filteredLinks = @($sharingLinksJson | ConvertFrom-Json)
                        
                        if ($filteredLinks.Count -eq 0) {
                            continue
                        }
                        
                        $script:Summary.SharingLinksFound += $filteredLinks.Count
                        
                        foreach ($link in $filteredLinks) {
                            [void]$script:Results.Add([PSCustomObject]@{
                                Library = $list.Title
                                ItemType = $itemType
                                ItemPath = $itemPath
                                SharingLinkId = $link.id
                                Scope = $link.link.scope
                                LinkUrl = $link.link.webUrl
                                Status = "Pending"
                            })
                        }
                        
                        if ($PSCmdlet.ShouldProcess($itemPath, "Delete $($filteredLinks.Count) sharing link(s)")) {
                            Write-Verbose "    Deleting sharing links from file: $itemPath"
                            
                            if ($Scope) {
                                $clearResult = m365 spo file sharinglink clear --webUrl $SiteUrl --fileUrl $itemPath --scope $Scope --force 2>&1
                            } else {
                                $clearResult = m365 spo file sharinglink clear --webUrl $SiteUrl --fileUrl $itemPath --force 2>&1
                            }
                            
                            if ($LASTEXITCODE -ne 0) {
                                Write-Warning "    Failed to clear sharing links for file '$itemPath'. CLI: $clearResult"
                                $script:Summary.Failed++
                                
                                $script:Results | Where-Object { $_.ItemPath -eq $itemPath -and $_.Status -eq "Pending" } | ForEach-Object {
                                    $_.Status = "Failed"
                                }
                                continue
                            }
                            
                            $script:Summary.SharingLinksDeleted += $filteredLinks.Count
                            Write-Host "    Deleted $($filteredLinks.Count) sharing link(s) from: $itemPath" -ForegroundColor Green
                            
                            $script:Results | Where-Object { $_.ItemPath -eq $itemPath -and $_.Status -eq "Pending" } | ForEach-Object {
                                $_.Status = "Deleted"
                            }
                        }
                    }
                    else {
                        $script:Summary.FoldersProcessed++
                        
                        Write-Verbose "    Checking folder: $itemPath"
                        $queryFilter = if ($Scope) { " --query `"[?link.scope == '$Scope']`"" } else { "" }
                        $sharingLinksJson = m365 spo folder sharinglink list --webUrl $SiteUrl --folderUrl $itemPath$queryFilter --output json 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "    Failed to list sharing links for folder '$itemPath'. CLI: $sharingLinksJson"
                            $script:Summary.Failed++
                            continue
                        }
                        
                        $filteredLinks = @($sharingLinksJson | ConvertFrom-Json)
                        
                        if ($filteredLinks.Count -eq 0) {
                            continue
                        }
                        
                        $script:Summary.SharingLinksFound += $filteredLinks.Count
                        
                        foreach ($link in $filteredLinks) {
                            [void]$script:Results.Add([PSCustomObject]@{
                                Library = $list.Title
                                ItemType = $itemType
                                ItemPath = $itemPath
                                SharingLinkId = $link.id
                                Scope = $link.link.scope
                                LinkUrl = $link.link.webUrl
                                Status = "Pending"
                            })
                        }
                        
                        if ($PSCmdlet.ShouldProcess($itemPath, "Delete $($filteredLinks.Count) sharing link(s)")) {
                            Write-Verbose "    Deleting sharing links from folder: $itemPath"
                            
                            if ($Scope) {
                                $clearResult = m365 spo folder sharinglink clear --webUrl $SiteUrl --folderUrl $itemPath --scope $Scope --force 2>&1
                            } else {
                                $clearResult = m365 spo folder sharinglink clear --webUrl $SiteUrl --folderUrl $itemPath --force 2>&1
                            }
                            
                            if ($LASTEXITCODE -ne 0) {
                                Write-Warning "    Failed to clear sharing links for folder '$itemPath'. CLI: $clearResult"
                                $script:Summary.Failed++
                                
                                $script:Results | Where-Object { $_.ItemPath -eq $itemPath -and $_.Status -eq "Pending" } | ForEach-Object {
                                    $_.Status = "Failed"
                                }
                                continue
                            }
                            
                            $script:Summary.SharingLinksDeleted += $filteredLinks.Count
                            Write-Host "    Deleted $($filteredLinks.Count) sharing link(s) from: $itemPath" -ForegroundColor Green
                            
                            $script:Results | Where-Object { $_.ItemPath -eq $itemPath -and $_.Status -eq "Pending" } | ForEach-Object {
                                $_.Status = "Deleted"
                            }
                        }
                    }
                }
                catch {
                    Write-Warning "    Error processing $itemType '$itemPath': $($_.Exception.Message)"
                    $script:Summary.Failed++
                    continue
                }
            }
        }
        catch {
            Write-Warning "Failed to process library '$($list.Title)': $($_.Exception.Message)"
            $script:Summary.Failed++
            continue
        }
    }
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Sharing Link Deletion Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Libraries Processed: $($script:Summary.LibrariesProcessed)" -ForegroundColor White
    Write-Host "Files Processed: $($script:Summary.FilesProcessed)" -ForegroundColor White
    Write-Host "Folders Processed: $($script:Summary.FoldersProcessed)" -ForegroundColor White
    Write-Host "Sharing Links Found: $($script:Summary.SharingLinksFound)" -ForegroundColor $(if ($script:Summary.SharingLinksFound -gt 0) { 'Yellow' } else { 'White' })
    
    if (-not $ReportOnly) {
        Write-Host "Sharing Links Deleted: $($script:Summary.SharingLinksDeleted)" -ForegroundColor $(if ($script:Summary.SharingLinksDeleted -gt 0) { 'Green' } else { 'White' })
    }
    
    Write-Host "Failed Operations: $($script:Summary.Failed)" -ForegroundColor $(if ($script:Summary.Failed -gt 0) { 'Red' } else { 'White' })
    Write-Host "========================================`n" -ForegroundColor Cyan
    
    if ($ExportToCsv -and $script:Results.Count -gt 0) {
        try {
            $script:Results | Export-Csv -Path $OutputPath -NoTypeInformation
            Write-Host "CSV report exported to: $OutputPath" -ForegroundColor Green
        }
        catch {
            Write-Warning "Failed to export CSV report: $($_.Exception.Message)"
        }
    }
    elseif ($ExportToCsv -and $script:Results.Count -eq 0) {
        Write-Host "No sharing links found. CSV report not created." -ForegroundColor Yellow
    }
}

# Example 1: Delete all sharing links with CSV export
# .\Delete-SharingLinks.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/demo" -ExportToCsv

# Example 2: Delete only anonymous sharing links
# .\Delete-SharingLinks.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/demo" -Scope anonymous

# Example 3: WhatIf mode (preview what would be deleted without actually deleting)
# .\Delete-SharingLinks.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/demo" -WhatIf

# Example 4: Verbose output with specific scope and CSV export
# .\Delete-SharingLinks.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/demo" -Scope users -Verbose -ExportToCsv
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
$siteUrl = Read-Host -Prompt "Enter site collection URL";
$dateTime = (Get-Date).toString("dd-MM-yyyy-hh-ss")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "SharingLinkDeletionReport-" + $dateTime + ".csv"
$ReportOutput = $directorypath + "\Logs\"+ $fileName

$global:Results = @();

#Exclude certain libraries
$ExcludedLists = @("Form Templates", "Preservation Hold Library", "Site Assets", "Images", "Pages", "Settings", "Videos","Timesheet"
    "Site Collection Documents", "Site Collection Images", "Style Library", "AppPages", "Apps for SharePoint", "Apps for Office")

Function QueryDeleteSharingLinksByObject($_web,$_object,$_type,$_relativeUrl,$_siteUrl,$_siteTitle,$_listTitle)
{
  $roleAssignments = Get-PnPProperty -ClientObject $_object -Property RoleAssignments
  
  foreach($roleAssign in $roleAssignments){
      Get-PnPProperty -ClientObject $roleAssign -Property RoleDefinitionBindings,Member;
      #Sharing link is in the format SharingLinks.03012675-2057-4d1d-91e0-8e3b176edd94.OrganizationView.20d346d3-d359-453b-900c-633c1551ccaa
    If ($roleAssign.Member.Title -like "SharingLinks*")
      {
        $global:Results += New-Object PSObject -property $([ordered]@{
            object= $_object.Title
            type = $_type          
            relativeURL = $_relativeURL
            siteUrl = $_siteUrl 
            siteTitle = $_siteTitle
            listTitle = $_listTitle 
            sharinglink = $roleAssign.Member.Title
        })
       Remove-PnPGroup -identity $roleAssign.Member.Title -force
    }
   }
}

  
Connect-PnPOnline -Url $siteUrl -Interactive

$web= Get-PnPWeb

Write-Host "Processing site $siteUrl"  -Foregroundcolor "Red"; 

$ll = Get-PnPList -Includes BaseType, Hidden, Title,HasUniqueRoleAssignments,RootFolder | Where-Object {$_.Hidden -eq $False -and $_.Title -notin $ExcludedLists } #$_.BaseType -eq "DocumentLibrary" 
  Write-Host "Number of lists $($ll.Count)";

  foreach($list in $ll)
  {
    $listUrl = $list.RootFolder.ServerRelativeUrl;       
    $listTitle = $list.Title; 
    #Get all list items in batches
    $ListItems = Get-PnPListItem -List $list -PageSize 2000 
        #Iterate through each list item
        ForEach($item in $ListItems)
        {
            $ItemCount = $ListItems.Count
            #Check if the Item has unique permissions
            $HasUniquePermissions = Get-PnPProperty -ClientObject $Item -Property "HasUniqueRoleAssignments"
            If($HasUniquePermissions)
            {       
                #Get Shared Links
                if($list.BaseType -eq "DocumentLibrary")
                {
                    $type= "File";
                    $fileUrl = $item.FieldValues.FileRef;
                }
                else
                {
                    $type= "Item";
                    $fileUrl = "$siteurl/lists/$listTitle/AllItems.aspx?FilterField1=ID&FilterValue1=$($item.id)"
                }
                QueryDeleteSharingLinksByObject $web $item $Type $fileUrl $siteUrl $web.Title $listTitle;
            }
        }
    }
 
  $global:Results | Export-CSV $ReportOutput -NoTypeInformation
Write-host -f Green "Sharing Links for user generated Successfully!"
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Deletion of sharing links with PowerShell](https://reshmeeauckloo.com/posts/powershell-delete-sharinglinks/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-delete-sharinglink-folder-file-item" aria-hidden="true" />

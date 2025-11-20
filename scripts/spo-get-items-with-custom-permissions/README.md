

# Find all items with unique permissions and export to csv

## Summary

It is a very common request to inventory the items with custom permissions.

![Example Screenshot](assets/example.png)


# [PnP PowerShell](#tab/pnpps)

```powershell

$adminSiteURL = "https://contoso-admin.sharepoint.com/"
$listOfItemWithCustomPermissionsCSVPath = "C:\temp\itemswithcustompermissions.csv"
$listOfListsWithCustomPermissionsCSVPath = "C:\temp\listswithcustompermissions.csv"

function Handle-Web ($webUrl)
{
    try 
    {
        # Most likely you should use the app-only approach, but for now I'm using the interactive approach
        #$localconn = Connect-PnPOnline -Url $webUrl -ClientId $ClientId -thumbprint $thumbprint -Tenant $TenantName -ReturnConnection -erroraction stop
        $localconn = Connect-PnPOnline -Url $webUrl -Interactive -ReturnConnection
        #first root
        $lists = Get-PnPList -Connection $localconn
        foreach($list in $lists)
        {
            $IsSystemList = Get-PnPProperty -ClientObject $list -Property IsSystemList -Connection $localconn
            if($IsSystemList)
            {
                write-host " Skipping $($list.Title) on $webUrl" -ForegroundColor Yellow
                continue  #skipping the system lists
            }
            write-host " handling $($list.Title) on $($webUrl)" -ForegroundColor Blue
            $listHasUniqueRoleAssignments = Get-PnPProperty -ClientObject $list -Property "HasUniqueRoleAssignments" -Connection $localconn
            if($listHasUniqueRoleAssignments )
            {
                $listInfo = New-Object PSObject
                $listInfo | Add-Member NoteProperty Title($list.Title) 
                $listInfo | Add-Member NoteProperty Url($list.ParentWebUrl) 
                $global:listOfListsWithCustomPermissions+=$listInfo
            }
            else
            {
                $listitems = Get-PnPListItem -List $list -PageSize 500 -Connection $localconn
                foreach($listItem in $listitems)
                {
                    $listItemHasUniqueRoleAssignments = Get-PnPProperty -ClientObject $listItem -Property HasUniqueRoleAssignments -Connection $localconn
                    if($listItemHasUniqueRoleAssignments)
                    {
                        $listItemInfo = New-Object PSObject
                        $listItemInfo | Add-Member NoteProperty Title($listItem["FileLeafRef"]) 
                        $listItemInfo | Add-Member NoteProperty List($list.Title)
                        $listItemInfo | Add-Member NoteProperty Url($list.ParentWebUrl)
                        $global:listOfItemWithCustomPermissions += $listItemInfo
                    }
                }
            }
        }  
        #then sub sites (which shouldn't be there ;-))
        $subs = Get-PnPSubWeb -Recurse -Connection $localconn
        foreach($sub in $subs)
        {
            Handle-Web -webUrl $sub.Url
        }      
    }
    catch 
    {
        write-host $_.Exception.Message
        #log the error
    }    
}



$global:listOfItemWithCustomPermissions = @()    
$global:listOfListsWithCustomPermissions = @()
$conn = Connect-PnPOnline -Url $adminSiteURL -Interactive -ReturnConnection
$allSites = Get-PnPTenantSite -Connection $conn
try 
{
    $counter = 0
    $allSitesCount = $allSites.Count
    foreach($site in $allSites)
    {
        write-host " at $counter of $allSitesCount" -ForegroundColor Green 
        $counter++
        
              
        #first root
        Handle-Web $site.Url
        
    }
        
}
catch 
{
    Write-Error $_.Exception.Message    
}

$listOfItemWithCustomPermissions | Export-Csv -Path $listOfItemWithCustomPermissionsCSVPath -Force -Encoding utf8BOM -Delimiter "|"
$listOfListsWithCustomPermissions | Export-Csv -Path $listOfListsWithCustomPermissionsCSVPath -Force -Encoding utf8BOM -Delimiter "|"

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


# [CLI for Microsoft 365 + PowerShell](#tab/cli-m365-ps)

```powershell
function Get-SpoItemsWithCustomPermissionsCli {
    [CmdletBinding(SupportsShouldProcess = $false)]
    param(
        [Parameter(Mandatory, HelpMessage = 'SharePoint tenant admin URL, e.g. https://contoso-admin.sharepoint.com.')]
        [ValidatePattern('^https://')]
        [string]$AdminUrl,

        [Parameter(Mandatory, HelpMessage = 'Path for the CSV report listing items that have unique permissions.')]
        [ValidateNotNullOrEmpty()]
        [string]$ItemReportPath,

        [Parameter(Mandatory, HelpMessage = 'Path for the CSV report listing lists that have unique permissions.')]
        [ValidateNotNullOrEmpty()]
        [string]$ListReportPath
    )

    begin {
        $script:Summary = [ordered]@{
            SitesProcessed             = 0
            WebsProcessed              = 0
            ListsProcessed             = 0
            ItemsWithUniquePermissions = 0
            ListsWithUniquePermissions = 0
            Failures                   = 0
        }

        foreach ($path in @($ItemReportPath, $ListReportPath)) {
            $directory = Split-Path -Path $path -Parent
            if ($directory -and -not (Test-Path -Path $directory)) {
                New-Item -ItemType Directory -Path $directory -Force | Out-Null
            }
        }

        Write-Host 'Ensuring Microsoft 365 CLI authentication.' -ForegroundColor Cyan
        m365 login --ensure | Out-Null

        $script:ItemResults = New-Object System.Collections.Generic.List[pscustomobject]
        $script:ListResults = New-Object System.Collections.Generic.List[pscustomobject]
    }

    process {
        Write-Host "Retrieving SharePoint sites from $AdminUrl." -ForegroundColor Cyan
        $tenantSitesOutput = m365 spo tenant site list --output json --query "[].{Url:Url}" 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list tenant sites. CLI output: $tenantSitesOutput"
        }

        try {
            $tenantSites = $tenantSitesOutput | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse tenant site response. $($_.Exception.Message)"
        }

        foreach ($site in $tenantSites) {
            $rootWebUrl = $site.Url
            $script:Summary.SitesProcessed++
            Write-Host "Processing site $rootWebUrl" -ForegroundColor Yellow

            $webQueue = [System.Collections.Generic.Queue[string]]::new()
            $webQueue.Enqueue($rootWebUrl)

            while ($webQueue.Count -gt 0) {
                $currentWebUrl = $webQueue.Dequeue()
                $script:Summary.WebsProcessed++
                Write-Host "  Processing web $currentWebUrl" -ForegroundColor Cyan

                $listsOutput = m365 spo list list --webUrl $currentWebUrl --properties Title,HasUniqueRoleAssignments,IsSystemList --output json --query "[?IsSystemList==\`false\`].{Title:Title,HasUniqueRoleAssignments:HasUniqueRoleAssignments}" 2>&1
                if ($LASTEXITCODE -ne 0) {
                    $script:Summary.Failures++
                    Write-Warning "Failed to list lists for web $currentWebUrl. CLI output: $listsOutput"
                    continue
                }

                try {
                    $lists = $listsOutput | ConvertFrom-Json -ErrorAction Stop
                }
                catch {
                    $script:Summary.Failures++
                    Write-Warning "Unable to parse lists for $currentWebUrl. $($_.Exception.Message)"
                    continue
                }

                foreach ($list in $lists) {
                    $script:Summary.ListsProcessed++

                    if ($list.HasUniqueRoleAssignments) {
                        $script:ListResults.Add([pscustomobject]@{
                                SiteUrl  = $currentWebUrl
                                ListName = $list.Title
                            }) | Out-Null
                        $script:Summary.ListsWithUniquePermissions++
                        continue
                    }

                    $itemsOutput = m365 spo listitem list --webUrl $currentWebUrl --listTitle $list.Title --fields FileLeafRef,FileDirRef,HasUniqueRoleAssignments --output json --query "[?HasUniqueRoleAssignments==\`true\`].{Item:FileLeafRef,Path:FileDirRef}" 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        $script:Summary.Failures++
                        Write-Warning "Failed to list items for list '$($list.Title)' in $currentWebUrl. CLI output: $itemsOutput"
                        continue
                    }

                    try {
                        $items = $itemsOutput | ConvertFrom-Json -ErrorAction Stop
                    }
                    catch {
                        $script:Summary.Failures++
                        Write-Warning "Unable to parse items for list '$($list.Title)' in $currentWebUrl. $($_.Exception.Message)"
                        continue
                    }

                    foreach ($item in $items) {
                        $script:ItemResults.Add([pscustomobject]@{
                                SiteUrl  = $currentWebUrl
                                ListName = $list.Title
                                ItemName = $item.Item
                                ItemPath = $item.Path
                            }) | Out-Null
                        $script:Summary.ItemsWithUniquePermissions++
                    }
                }

                $subWebsOutput = m365 spo web list --url $currentWebUrl --output json --query "[].{Url:Url}" 2>&1
                if ($LASTEXITCODE -ne 0) {
                    $script:Summary.Failures++
                    Write-Warning "Failed to list subsites for $currentWebUrl. CLI output: $subWebsOutput"
                    continue
                }

                if ($subWebsOutput.Trim()) {
                    try {
                        $subWebs = $subWebsOutput | ConvertFrom-Json -ErrorAction Stop
                    }
                    catch {
                        $script:Summary.Failures++
                        Write-Warning "Unable to parse subsites for $currentWebUrl. $($_.Exception.Message)"
                        continue
                    }

                    foreach ($sub in $subWebs) {
                        if ($sub.Url) {
                            $webQueue.Enqueue($sub.Url)
                        }
                    }
                }
            }
        }
    }

    end {
        Write-Host 'Writing CSV reports.' -ForegroundColor Cyan
        $script:ItemResults | Export-Csv -Path $ItemReportPath -NoTypeInformation -Encoding UTF8
        $script:ListResults | Export-Csv -Path $ListReportPath -NoTypeInformation -Encoding UTF8

        Write-Host 'Summary' -ForegroundColor Green
        foreach ($entry in $script:Summary.GetEnumerator()) {
            Write-Host (" - {0}: {1}" -f $entry.Key, $entry.Value)
        }
    }
}

# Example usage
Get-SpoItemsWithCustomPermissionsCli \
    -AdminUrl "https://contoso-admin.sharepoint.com" \
    -ItemReportPath "C:\\Temp\\ItemsWithUniquePermissions.csv" \
    -ListReportPath "C:\\Temp\\ListsWithUniquePermissions.csv"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |
| Adam Wójcik (Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-items-with-custom-permissions" aria-hidden="true" />

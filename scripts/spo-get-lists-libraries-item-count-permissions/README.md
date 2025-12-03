

# Export all List and Libraries with Item count and Permission in CSV

## Summary
Get all lists and Libraries along with total Item count and permissions and export it in CSV file using below power shell script.

![PnP Powershell result](assets/PnPPowershellExample.png)

----
CLI version of the script works the same but does not retrieve permissions information from the list

![PnP Powershell result](assets/M365CLIExample.png)

# [PnP PowerShell](#tab/pnpps)
```powershell

# Make sure necessary modules are installed
# PnP PowerShell to get access to M365 tenent

Install-Module PnP.PowerShell
$siteURL = "https://tenent.sharepoint.com/sites/Dataverse"
$ReportOutput="C:\SiteInventory.csv"
$ResultData = @()
$UniquePermission = "";
#  -UseWebLogin used for 2 factor Auth.  You can remove if you don't have MFA turned on
Connect-PnPOnline -Url  $siteUrl
 # get all lists from given SharePoint Site collection
 $lists =  Get-PnPList -Includes HasUniqueRoleAssignments,RoleAssignments
 If($lists.Count -gt 0){
   foreach($list in $lists){
    $members = "";
    if($list.HasUniqueRoleAssignments -eq $false){
        $UniquePermission = "Inherited"
    }
    if($list.HasUniqueRoleAssignments -eq $true){
        $UniquePermission = "Unique"    
    }
    if($list.RoleAssignments.Count -gt 0){
        foreach($roleAssignment in $list.RoleAssignments){
            $property = Get-PnPProperty -ClientObject $roleAssignment -Property Member
            $members += $property.Title + ";"
        }
    }
     $ResultData+= New-Object PSObject -Property @{
            'List-Library Name' = $list.Title;
            'Id'=$list.Id;
            'Parent Web URL'=$list.ParentWebUrl;
            'Item Count' = $list.ItemCount;
            'Last Modified' = $list.LastItemUserModifiedDate.ToString();
            'Created'=$list.Created;
            'Default View URL'=$list.DefaultViewUrl;
            'Permision'=$UniquePermission;
            'Members'=$members;
            'isHidden'=$list.Hidden;
        }
   }
 }

 $ResultData | Export-Csv $ReportOutput -NoTypeInformation
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]


# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
function Get-SpoListInventory {
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter(Mandatory = $true, HelpMessage = 'URL of the SharePoint site (e.g., https://contoso.sharepoint.com/sites/Project).')]
        [string]$SiteUrl,

        [Parameter(Mandatory = $false, HelpMessage = 'Optional file path where the CSV export should be saved.')]
        [string]$OutputCsvPath,

        [Parameter(Mandatory = $false, HelpMessage = 'Include hidden lists/libraries in the output.')]
        [switch]$IncludeHidden,

        [Parameter(Mandatory = $false, HelpMessage = 'Bypass confirmation prompts when exporting to CSV.')]
        [switch]$Force
    )

    begin {
        Write-Verbose "Ensuring authentication for $SiteUrl"
        m365 login --ensure
        $script:Result = [System.Collections.Generic.List[object]]::new()
        $script:Stats = [ordered]@{
            TotalLists    = 0
            HiddenSkipped = 0
            Exported      = 0
        }
    }

    process {
        Write-Verbose "Retrieving lists from $SiteUrl"
        $listResponse = m365 spo list list --webUrl $SiteUrl --properties "Title,Id,ParentWebUrl,ItemCount,LastItemUserModifiedDate,Created,DefaultViewUrl,Hidden" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve lists. CLI output: $listResponse"
        }

        try {
            $lists = $listResponse | ConvertFrom-Json
        }
        catch {
            throw "Unable to parse list response as JSON. Error: $($_.Exception.Message)"
        }

        foreach ($list in @($lists)) {
            $script:Stats.TotalLists++

            if (-not $IncludeHidden -and $list.Hidden) {
                $script:Stats.HiddenSkipped++
                continue
            }

            $record = [PSCustomObject]@{
                ListName       = $list.Title
                Id             = $list.Id
                ParentWebUrl   = $list.ParentWebUrl
                ItemCount      = $list.ItemCount
                LastModified   = $list.LastItemUserModifiedDate
                Created        = $list.Created
                DefaultViewUrl = $list.DefaultViewUrl
                Hidden         = [bool]$list.Hidden
            }

            $script:Result.Add($record)
            Write-Verbose "Processed list '$($list.Title)'"
        }
    }

    end {
        Write-Host "Lists processed: $($script:Stats.TotalLists)"
        if (-not $IncludeHidden) {
            Write-Host "Hidden lists skipped: $($script:Stats.HiddenSkipped)"
        }

        if ($OutputCsvPath) {
            $shouldExport = $Force -or $PSCmdlet.ShouldProcess($OutputCsvPath, 'Export list inventory to CSV')

            if ($shouldExport) {
                try {
                    $directory = Split-Path -Path $OutputCsvPath -Parent
                    if ($directory -and -not (Test-Path -Path $directory)) {
                        Write-Verbose "Creating directory $directory"
                        New-Item -ItemType Directory -Path $directory -Force | Out-Null
                    }

                    $script:Result | Export-Csv -Path $OutputCsvPath -NoTypeInformation -Encoding UTF8
                    $script:Stats.Exported = $script:Result.Count
                    Write-Host "CSV exported to $OutputCsvPath"
                }
                catch {
                    throw "Failed to export CSV. Error: $($_.Exception.Message)"
                }
            }
        }
        else {
            $script:Result
        }

        if ($script:Stats.Exported -gt 0) {
            Write-Host "Records exported: $($script:Stats.Exported)"
        }
    }
}

# Example usage:
# Get-SpoListInventory -SiteUrl "https://contoso.sharepoint.com/sites/Dataverse" -OutputCsvPath "C:\\Reports\\SiteInventory.csv" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Dipen Shah](https://github.com/dips365) |
| [Adam Wójcik](https://github.com/Adam-it)|
| [Alex Talarico](https://github.com/getalex) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/bulk-undelete-from-recyclebin" aria-hidden="true" />

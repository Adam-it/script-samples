

# Undelete items from SharePoint Recycle bin

## Summary
sometimes users need to restore items from SharePoint recycle bin. This script allows them to undelete items from recycle bin and restore it in respective document library and list.

# [PnP PowerShell](#tab/pnpps)
```powershell

# Make sure necessary modules are installed
# PnP PowerShell to get access to M365 tenent

Install-Module PnP.PowerShell
$siteURL = "https://tenent.sharepoint.com/sites/Dataverse"
$rows = 10000 
$userEmailAddress = "user@tenent.onmicrosoft.com" #admin user
#  -UseWebLogin used for 2 factor Auth.  You can remove if you don't have MFA turned on
Connect-PnPOnline -Url  $siteUrl
 $deletedItems = $null
 # Get files which is deleted by specific user.
 $deletedItems = Get-PnPRecycleBinItem -FirstStage -RowLimit $rows | Where-Object {$_.DeletedByEmail -Eq $userEmailAddress} | select Id,Title,LeafName,ItemType
 if($deletedItems.Count -gt 0)
 {
    Foreach ($deletedItem in $deletedItems){
        Write-Host "Restoring is in process for Item Id : " $deletedItem.Id
        Restore-PnPRecycleBinItem -Identity $deletedItem.Id.ToString() -Force
        Write-Host "Item with Id : " $deletedItem.Id " has been restored successfully."
    }
 }

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site recycle bin to inspect.")]
    [string]$SiteUrl,

    [Parameter(HelpMessage = "Filter results to items deleted by this email address.")]
    [string]$DeletedByEmail,

    [switch]$IncludeSecondStage
)

begin {
    m365 login --ensure

    $script:CollectedItems = [System.Collections.Generic.List[psobject]]::new()
    $script:Summary = [ordered]@{
        Site              = $SiteUrl
        StagesQueried     = if ($IncludeSecondStage) { 'FirstStage,SecondStage' } else { 'FirstStage' }
        DeletedByFilter   = if ($DeletedByEmail) { $DeletedByEmail } else { 'None' }
        ItemsFound        = 0
        ItemsRestored     = 0
        ItemsFailed       = 0
        ItemsSimulated    = 0
    }

    Write-Host "Gathering recycle bin items from $SiteUrl"
    if ($DeletedByEmail) {
        Write-Host "Filtering to items deleted by $DeletedByEmail"
    }
}

process {
    $stages = if ($IncludeSecondStage) { 'FirstStage', 'SecondStage' } else { 'FirstStage' }

    foreach ($stage in $stages) {
        Write-Host "Retrieving $stage items..."
        $listArgs = @('spo', 'site', 'recyclebinitem', 'list', '--siteUrl', $SiteUrl, '--stage', $stage, '--output', 'json')
        if ($DeletedByEmail) {
            $listArgs += @('--query', "[?DeletedByEmail == '$DeletedByEmail']")
        }

        $listOutput = & m365 @listArgs 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to list $stage items. CLI: $listOutput"
            continue
        }

        $items = if ([string]::IsNullOrWhiteSpace($listOutput)) { @() } else { @($listOutput | ConvertFrom-Json) }
        foreach ($item in $items) {
            $CollectedItems.Add([pscustomobject]@{
                Stage          = $stage
                Id             = $item.Id
                Title          = $item.Title
                ItemType       = $item.ItemType
                DeletedByEmail = $item.DeletedByEmail
            })
        }
    }
}

end {
    if ($CollectedItems.Count -eq 0) {
        Write-Host "No recycle bin items matched the provided criteria. Nothing to restore."
        return
    }

    $uniqueItems = $CollectedItems | Sort-Object Id -Unique
    $Summary.ItemsFound = $uniqueItems.Count

    Write-Host "Items ready for restore ($($uniqueItems.Count)):"
    foreach ($item in $uniqueItems) {
        Write-Host "  [$($item.Stage)] Id=$($item.Id) Title=$($item.Title)"
    }

    $ids = ($uniqueItems | Select-Object -ExpandProperty Id) -join ','
    $actionDescription = "Restore {0} recycle bin item(s)" -f $uniqueItems.Count

    if (-not $PSCmdlet.ShouldProcess($SiteUrl, $actionDescription)) {
        $Summary.ItemsSimulated = $uniqueItems.Count
        Write-Host "WhatIf: restore skipped."
    } else {
        Write-Host "Submitting restore request..."
        $restoreArgs = @('spo', 'site', 'recyclebinitem', 'restore', '--siteUrl', $SiteUrl, '--ids', $ids, '--output', 'json')
        $restoreOutput = & m365 @restoreArgs 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to restore items. CLI: $restoreOutput"
            $Summary.ItemsFailed = $uniqueItems.Count
        } else {
            $Summary.ItemsRestored = $uniqueItems.Count
            Write-Host "Restore complete."
        }
    }

    Write-Host ("Summary: {0} items found, {1} restored, {2} simulated, {3} failed." -f `
        $Summary.ItemsFound, $Summary.ItemsRestored, $Summary.ItemsSimulated, $Summary.ItemsFailed)
}
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Dipen Shah](https://github.com/dips365) |
| [Adam Wójcik](https://github.com/Adam-it)|


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/bulk-undelete-from-recyclebin" aria-hidden="true" />

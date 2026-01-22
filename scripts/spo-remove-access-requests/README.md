

# Remove site access requests

## Summary

Sometimes, as a site owner you cannot manage all site access requests for your site. Especially when users request access for specific content on the site (Site page, list etc.) you might prefer to grant access to the entire site. Use PnP PowerShell or CLI for Microsoft 365 to remove site access requests depending on their status.
 
![Example Screenshot](assets/example.png)


# [PnP PowerShell](#tab/pnpps)

```powershell

$siteUrl = "https://contoso.sharepoint.com/sites/DemoSite"

# Request status: 0 - Pending, 1 - Accepted, 3 - Declined
$requestStatus = 0

Connect-PnPOnline -Url $siteUrl -Interactive

$batch = New-PnPBatch
$accessRequestsList = Get-PnPList | Where-Object {$_.Title -eq "Access Requests"}
$itemsToRemove = Get-PnPListItem -List $accessRequestsList -Query "<View><Query><Where><And><Eq><FieldRef Name='Status'/><Value Type='Number'>$requestStatus</Value></Eq><Eq><FieldRef Name='IsInvitation'/><Value Type='Boolean'>0</Value></Eq></And></Where></Query></View>
"
$itemsToRemove | ForEach-Object { 
    Remove-PnPListItem -List $accessRequestsList -Identity $_.Id -Batch $batch
}
Invoke-PnPBatch -Batch $batch

Disconnect-PnPOnline

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]


# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "SharePoint site URL (e.g., 'https://contoso.sharepoint.com/sites/project')")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)(/.*)?$')]
    [string]$SiteUrl,
    
    [Parameter(HelpMessage = "Request status to remove: 0=Pending, 1=Accepted, 3=Declined")]
    [ValidateSet(0, 1, 3)]
    [int]$RequestStatus = 0,
    
    [Parameter(HelpMessage = "Permanently delete items instead of moving to recycle bin")]
    [switch]$HardDelete
)

begin {
    $script:Summary = @{
        Found = 0
        Removed = 0
        Failures = 0
    }
    
    $transcriptPath = "$((Get-Location).Path)/RemoveAccessRequests-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Remove Access Requests - CLI for Microsoft 365" -ForegroundColor Cyan
    Write-Host "==============================================" -ForegroundColor Cyan
    Write-Host ""
    
    Write-Verbose "Ensuring Microsoft 365 login..."
    m365 login --ensure | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure Microsoft 365 login. Please run 'm365 login' first."
    }
    Write-Host "✓ Authenticated successfully" -ForegroundColor Green
    Write-Host ""
    
    Write-Host "Target Site: $SiteUrl" -ForegroundColor White
    $statusText = switch ($RequestStatus) {
        0 { "Pending" }
        1 { "Accepted" }
        3 { "Declined" }
    }
    Write-Host "Request Status: $statusText ($RequestStatus)" -ForegroundColor White
    Write-Host "Deletion Mode: $(if ($HardDelete) { 'Permanent' } else { 'Recycle Bin' })" -ForegroundColor White
    Write-Host ""
    
    Write-Verbose "Finding Access Requests list..."
    $listsJson = m365 spo list list --webUrl $SiteUrl --filter "Title eq 'Access Requests'" --output json 2>&1
    
    if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve lists from site. Error: $listsJson"
        Write-Host ""
        Write-Host "Site may not exist or you may not have permissions." -ForegroundColor Yellow
        Write-Host "Exiting gracefully..." -ForegroundColor Gray
        Stop-Transcript
        return
    }
    
    $lists = @($listsJson | ConvertFrom-Json)
    $accessRequestsList = $lists | Where-Object { $_.Title -eq 'Access Requests' }
    
    if ($null -eq $accessRequestsList) {
        Write-Host "Access Requests list not found on this site." -ForegroundColor Yellow
        Write-Host ""
        Write-Host "This is normal for sites that haven't had any access requests." -ForegroundColor Gray
        Write-Host "Exiting gracefully..." -ForegroundColor Gray
        Stop-Transcript
        return
    }
    
    Write-Host "✓ Found Access Requests list (ID: $($accessRequestsList.Id))" -ForegroundColor Green
    Write-Host ""
    
    $script:AccessRequestsListTitle = $accessRequestsList.Title
}

process {
    Write-Host "Querying access requests..." -ForegroundColor Cyan
    Write-Host ""
    
    $camlQuery = "<View><Query><Where><And><Eq><FieldRef Name='Status'/><Value Type='Number'>$RequestStatus</Value></Eq><Eq><FieldRef Name='IsInvitation'/><Value Type='Boolean'>0</Value></Eq></And></Where></Query></View>"
    
    Write-Verbose "CAML Query: $camlQuery"
    
    try {
        $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listTitle $script:AccessRequestsListTitle --camlQuery $camlQuery --output json
        if ($LASTEXITCODE -ne 0) {
            throw "CLI error: $itemsJson"
        }
        
        $items = @($itemsJson | ConvertFrom-Json)
        $script:Summary.Found = $items.Count
        
        if ($items.Count -eq 0) {
            Write-Host "No access requests found with status '$statusText'." -ForegroundColor Yellow
            Write-Host ""
            Write-Host "Nothing to remove." -ForegroundColor Gray
            return
        }
        
        Write-Host "Found $($items.Count) access request(s) with status '$statusText':" -ForegroundColor White
        Write-Host ""
        
        $items | ForEach-Object {
            Write-Host "  - ID: $($_.Id) | Title: $($_.Title) | Created: $($_.Created)" -ForegroundColor Gray
        }
        
        Write-Host ""
        
        $action = if ($HardDelete) { "Permanently delete" } else { "Move to recycle bin" }
        if ($PSCmdlet.ShouldProcess("$($items.Count) access request(s)", $action)) {
            Write-Host "Removing access requests..." -ForegroundColor Yellow
            
            $ids = ($items | ForEach-Object { $_.Id }) -join ','
            Write-Verbose "Item IDs: $ids"
            
            try {
                if ($HardDelete) {
                    $removeResult = m365 spo listitem batch remove --webUrl $SiteUrl --listTitle $script:AccessRequestsListTitle --ids $ids --force 2>&1
                }
                else {
                    $removeResult = m365 spo listitem batch remove --webUrl $SiteUrl --listTitle $script:AccessRequestsListTitle --ids $ids --recycle --force 2>&1
                }
                
                if ($LASTEXITCODE -ne 0) {
                    throw "CLI error: $removeResult"
                }
                
                $script:Summary.Removed = $items.Count
                Write-Host "✓ Successfully removed $($items.Count) access request(s)" -ForegroundColor Green
            }
            catch {
                Write-Warning "Failed to remove access requests: $_"
                $script:Summary.Failures++
            }
        }
        else {
            Write-Host "WhatIf: Would $($action.ToLower()) $($items.Count) access request(s)" -ForegroundColor Cyan
            $script:Summary.Removed = $items.Count
        }
    }
    catch {
        Write-Warning "Failed to query access requests: $_"
        $script:Summary.Failures++
    }
}

end {
    Write-Host ""
    Write-Host "======================" -ForegroundColor Cyan
    Write-Host "Summary" -ForegroundColor Cyan
    Write-Host "======================" -ForegroundColor Cyan
    Write-Host ""
    
    Write-Host "Access Requests Found:   $($script:Summary.Found)" -ForegroundColor $(if ($script:Summary.Found -gt 0) { 'White' } else { 'Gray' })
    Write-Host "Access Requests Removed: $($script:Summary.Removed)" -ForegroundColor $(if ($script:Summary.Removed -gt 0) { 'Green' } else { 'Gray' })
    Write-Host "Failures:                $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    
    Write-Host ""
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Gray
    
    Stop-Transcript
}

# Example 1: Remove all pending access requests (move to recycle bin)
# .\Remove-AccessRequests.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project"

# Example 2: Permanently delete accepted access requests
# .\Remove-AccessRequests.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -RequestStatus 1 -HardDelete

# Example 3: Test with WhatIf before removing declined requests
# .\Remove-AccessRequests.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -RequestStatus 3 -WhatIf

# Example 4: Run with verbose output to see CAML query details
# .\Remove-AccessRequests.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Adam Wójcik |
| [Aimery Thomas](https://github.com/a1mery)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-remove-access-requests" aria-hidden="true" />

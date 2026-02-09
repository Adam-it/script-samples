

# Get, Update, Add, Remove SharePoint list items in large lists

## Summary

Working and processing lists items in large lists.
PnP PowerShell and CLI for Microsoft 365 examples

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Copy a script  below,

# [PnP PowerShell](#tab/pnpps)
```powershell

$url = "https://yourtenantname.sharepoint.com/sites/SiteCollection"
$list = "YourLargeList"
Connect-PnPOnline -Url $Url -Interactive


# create 5000+ list items
$batch = New-PnPBatch
1..5500 | ForEach-Object { 
            Add-PnPListItem -List $list -Values @{"Title"="Test Item Batched $_"} -Batch $batch 
           }

Invoke-PnPBatch -Batch $batch


#Update each list item separatelly
$batch = New-PnPBatch
$items = Get-PnPListItem -List $list -PageSize 1000
$items | ForEach-Object { 
            
            Set-PnPListItem -List $list -Identity $_.Id -Values @{"Title"="Test Item Batched and updated $_"} -Batch $batch
           }

Invoke-PnPBatch -Batch $batch


#remove each list item separatelly
$batch = New-PnPBatch
$items = Get-PnPListItem -List $list -PageSize 1000
$items | ForEach-Object { 
            Remove-PnPListItem -List $list -Identity $_.Id
           }

Invoke-PnPBatch -Batch $batch


#read each list item separatelly
$batch = New-PnPBatch
Get-PnPListItem -List $list -PageSize 1000 | ForEach-Object { 
            get-PnPListItem -List $list -Identity $_
           }

Invoke-PnPBatch -Batch $batch


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell

[CmdletBinding()]
param(
  [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site")]
  [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
  [string]$SiteUrl,

  [Parameter(Mandatory, HelpMessage = "Title of the SharePoint list")]
  [ValidateNotNullOrEmpty()]
  [string]$ListName,

  [Parameter(HelpMessage = "Number of items per page (100-5000)")]
  [ValidateRange(100, 5000)]
  [int]$PageSize = 1000,

  [Parameter(HelpMessage = "Path where transcript log will be saved")]
  [ValidateNotNullOrEmpty()]
  [string]$OutputPath = (Get-Location).Path
)

begin {
  $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
  $transcriptPath = Join-Path $OutputPath "LargeList-Operations-$timestamp.log"
  Start-Transcript -Path $transcriptPath

  Write-Host "Connecting to Microsoft 365..." -ForegroundColor Yellow
  m365 login --ensure
  if ($LASTEXITCODE -ne 0) {
    Stop-Transcript
    throw "Failed to authenticate to Microsoft 365"
  }

  Write-Host "Getting list properties..." -ForegroundColor Yellow
  $listPropertiesJson = m365 spo list get --title $ListName --webUrl $SiteUrl --output json
  if ($LASTEXITCODE -ne 0) {
    Stop-Transcript
    throw "Failed to get list properties for '$ListName'"
  }

  $listProperties = $listPropertiesJson | ConvertFrom-Json
  $itemCount = $listProperties.ItemCount
  $pageNumber = [int][Math]::Ceiling($itemCount / $PageSize)

  Write-Host "Found $itemCount items in list '$ListName'" -ForegroundColor Cyan
  Write-Host "Will process in $pageNumber pages (PageSize: $PageSize)`n" -ForegroundColor Cyan

  $script:Summary = @{
    TotalItems    = $itemCount
    ItemsAdded    = 0
    ItemsUpdated  = 0
    ItemsRemoved  = 0
    Failures      = 0
  }
}

process {
  Write-Host "`n========== OPERATION 1: Get All Items =========" -ForegroundColor Magenta
  Write-Host "Retrieving all items from large list...`n" -ForegroundColor Yellow

  for ($i = 0; $i -lt $pageNumber; $i++) {
    try {
      $pagePercent = [math]::Round((($i + 1) / $pageNumber) * 100, 1)
      Write-Host "Processing page $($i + 1)/$pageNumber ($pagePercent%)..." -ForegroundColor Gray

      $items = m365 spo listitem list --title $ListName --webUrl $SiteUrl --pageSize $PageSize --pageNumber $i --output json
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve items from page $($i + 1)"
        $script:Summary.Failures++
        continue
      }

      $itemsArray = $items | ConvertFrom-Json
      Write-Host "  Retrieved $($itemsArray.Count) items from page $($i + 1)" -ForegroundColor Green
    }
    catch {
      Write-Warning "Error processing page $($i + 1): $_"
      $script:Summary.Failures++
      continue
    }
  }

  Write-Host "`n========== OPERATION 2: Create List Items =========" -ForegroundColor Magenta
  Write-Host "Creating 100 demo items...`n" -ForegroundColor Yellow

  $itemsToCreate = 100
  $createdCount = 0

  1..$itemsToCreate | ForEach-Object {
    try {
      $itemNumber = $_
      if ($itemNumber % 10 -eq 0) {
        Write-Host "Creating items: $itemNumber/$itemsToCreate..." -ForegroundColor Gray
      }

      m365 spo listitem add --contentType Item --listTitle $ListName --webUrl $SiteUrl --Title "Demo Item $itemNumber using CLI" --output json | Out-Null
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to create item $itemNumber"
        $script:Summary.Failures++
      }
      else {
        $createdCount++
      }
    }
    catch {
      Write-Warning "Error creating item $itemNumber: $_"
      $script:Summary.Failures++
    }
  }

  $script:Summary.ItemsAdded = $createdCount
  Write-Host "Successfully created $createdCount items" -ForegroundColor Green

  Write-Host "`n========== OPERATION 3: Update List Items =========" -ForegroundColor Magenta
  Write-Host "Updating all items in list...`n" -ForegroundColor Yellow

  $updatedCount = 0
  for ($i = 0; $i -lt $pageNumber; $i++) {
    try {
      $pagePercent = [math]::Round((($i + 1) / $pageNumber) * 100, 1)
      Write-Host "Updating page $($i + 1)/$pageNumber ($pagePercent%)..." -ForegroundColor Gray

      $itemsJson = m365 spo listitem list --title $ListName --webUrl $SiteUrl --fields "ID" --pageSize $PageSize --pageNumber $i --output json
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve items from page $($i + 1) for update"
        $script:Summary.Failures++
        continue
      }

      $items = $itemsJson | ConvertFrom-Json
      $items | Select-Object -ExpandProperty ID | ForEach-Object {
        try {
          $itemId = $_
          m365 spo listitem set --listTitle $ListName --id $itemId --webUrl $SiteUrl --Title "Updated with CLI at $timestamp" --output json | Out-Null
          if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to update item ID $itemId"
            $script:Summary.Failures++
          }
          else {
            $updatedCount++
          }
        }
        catch {
          Write-Warning "Error updating item ID $itemId: $_"
          $script:Summary.Failures++
        }
      }
    }
    catch {
      Write-Warning "Error processing page $($i + 1) for update: $_"
      $script:Summary.Failures++
      continue
    }
  }

  $script:Summary.ItemsUpdated = $updatedCount
  Write-Host "Successfully updated $updatedCount items" -ForegroundColor Green

  Write-Host "`n========== OPERATION 4: Remove List Items =========" -ForegroundColor Magenta
  Write-Host "Removing all items from list...`n" -ForegroundColor Yellow

  $removedCount = 0
  for ($i = 0; $i -lt $pageNumber; $i++) {
    try {
      $pagePercent = [math]::Round((($i + 1) / $pageNumber) * 100, 1)
      Write-Host "Removing page $($i + 1)/$pageNumber ($pagePercent%)..." -ForegroundColor Gray

      $itemsJson = m365 spo listitem list --title $ListName --webUrl $SiteUrl --fields "ID" --pageSize $PageSize --pageNumber $i --output json
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve items from page $($i + 1) for removal"
        $script:Summary.Failures++
        continue
      }

      $items = $itemsJson | ConvertFrom-Json
      $items | Select-Object -ExpandProperty ID | ForEach-Object {
        try {
          $itemId = $_
          m365 spo listitem remove --webUrl $SiteUrl --listTitle $ListName --id $itemId --force
          if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to remove item ID $itemId"
            $script:Summary.Failures++
          }
          else {
            $removedCount++
          }
        }
        catch {
          Write-Warning "Error removing item ID $itemId: $_"
          $script:Summary.Failures++
        }
      }
    }
    catch {
      Write-Warning "Error processing page $($i + 1) for removal: $_"
      $script:Summary.Failures++
      continue
    }
  }

  $script:Summary.ItemsRemoved = $removedCount
  Write-Host "Successfully removed $removedCount items" -ForegroundColor Green
}

end {
  Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
  Write-Host "Site URL: $SiteUrl" -ForegroundColor White
  Write-Host "List Name: $ListName" -ForegroundColor White
  Write-Host "Page Size: $PageSize" -ForegroundColor White
  Write-Host "Total Items Found: $($Summary.TotalItems)" -ForegroundColor Green
  Write-Host "Items Added: $($Summary.ItemsAdded)" -ForegroundColor Green
  Write-Host "Items Updated: $($Summary.ItemsUpdated)" -ForegroundColor Green
  Write-Host "Items Removed: $($Summary.ItemsRemoved)" -ForegroundColor Green
  Write-Host "Failures: $($Summary.Failures)" -ForegroundColor $(if ($Summary.Failures -gt 0) { 'Red' } else { 'Green' })
  Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Cyan

  Stop-Transcript
}

# Example 1: Basic usage (WhatIf mode for get operations)
# .\script.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListName "LargeList"

# Example 2: Custom page size for better performance
# .\script.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListName "LargeList" -PageSize 2000

# Example 3: Custom output path for transcript
# .\script.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListName "LargeList" -OutputPath "C:\Logs"

# Example 4: Verbose mode for detailed execution
# .\script.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListName "LargeList" -Verbose


```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Valeras Narbutas |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-list-items-large-lists" aria-hidden="true" />

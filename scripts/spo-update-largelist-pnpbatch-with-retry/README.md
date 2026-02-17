

# Update large list with PnP-Batch with retries to address throttling challenges 

## Summary

This sample shows how to efficiently update a large SharePoint list of 60,000 or more items. It addresses throttling challenges, emphasizing exception handling and retry mechanisms to ensure smooth updates. Available in both PnP PowerShell and CLI for Microsoft 365 v11.4.0+.
 
# [PnP PowerShell](#tab/pnpps)

```PowerShell

$siteUrl = "https://contoso.sharepoint.com/teams/app-test"

Connect-PnPOnline –Url $siteUrl -Interactive

function UpdateType ($TypeColumn, $list) {
  do {
    try {
      $StopLoop = $false
      $batch = New-PnPBatch
      $index = 1; 
      $itemId = 0; 
      $listItems = Get-PnPListItem -List $list  -PageSize 500 | Where {$_.FieldValues.$TypeColumn -ne $null }
      $totalCount =  $listItems.Count

      $listItems| ForEach-Object {
        $itemId = $_.Id
        Set-PnPListItem -List $list -Identity $_.Id -Values @{$TypeColumn = $null;} -UpdateType SystemUpdate -Batch $batch

        if ($index % 100 -eq 0 -or $index -eq $listItems.Count) {
          write-host "Updating batch starting $index out of $totalCount on library $list"
          Invoke-PnPBatch $batch
          $batch = New-PnPBatch
        }
        $index+=1;
      }

      Write-Host "Job completed"
      $Stoploop = $true
    }
    catch {
      if ($Retrycount -gt 3) {
        Write-Host "Could not send Information after 3 retrys.$itemId after number of item  processed $index"
        $Stoploop = $true
      }
      else {
        Write-Host "Could not send Information retrying in 30 seconds...{$itemId} after number of item  processed {$index}"
        Start-Sleep -Seconds 30
        Connect-PnPOnline –Url $siteUrl -interactive
        $Retrycount = $Retrycount + 1
      }
    }
  }
  While ($Stoploop -eq $false)

  write-host $("End time " + (Get-Date) + " Updating column: " +  $TypeColumn + "from list " + $listName )
}

UpdateType "Type" "List1" 

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365)

```PowerShell

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage="SharePoint site URL where the list resides")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory, HelpMessage="Title of the SharePoint list to update")]
    [string]$ListName,
    
    [Parameter(Mandatory, HelpMessage="Internal name of the field to clear/set to null")]
    [string]$FieldName,
    
    [Parameter(HelpMessage="Maximum number of retry attempts per item (default: 3)")]
    [ValidateRange(1, 10)]
    [int]$MaxRetries = 3,
    
    [Parameter(HelpMessage="Delay in seconds between retry attempts (default: 30)")]
    [ValidateRange(10, 300)]
    [int]$RetryDelaySeconds = 30,
    
    [Parameter(HelpMessage="Output path for CSV report (default: current directory)")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $startTime = Get-Date
    $transcriptPath = Join-Path $OutputPath "ListUpdate_$(Get-Date -Format 'yyyyMMdd_HHmmss')_Transcript.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "=== Large List Field Update Script ===" -ForegroundColor Cyan
    Write-Host "Site: $SiteUrl" -ForegroundColor White
    Write-Host "List: $ListName" -ForegroundColor White
    Write-Host "Field: $FieldName" -ForegroundColor White
    Write-Host "Max Retries: $MaxRetries" -ForegroundColor White
    Write-Host "Retry Delay: $RetryDelaySeconds seconds" -ForegroundColor White
    Write-Host ""
    
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath)) {
            throw "Output path does not exist: $OutputPath"
        }
    }
    
    Write-Verbose "Authenticating to Microsoft 365..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate to Microsoft 365"
    }
    Write-Verbose "Authentication successful"
    
    $script:ReportCollection = [System.Collections.ArrayList]::new()
    $script:Summary = @{
        ItemsFound = 0
        Updated = 0
        Failures = 0
    }
    
    Write-Host "Retrieving items from list '$ListName' where $FieldName is not null..." -ForegroundColor Cyan
    Write-Verbose "Executing: m365 spo listitem list --webUrl $SiteUrl --listTitle $ListName --fields 'ID,$FieldName' --filter '$FieldName ne null' --output json"
    
    try {
        $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listTitle $ListName --fields "ID,$FieldName" --filter "$FieldName ne null" --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve list items"
        }
        
        $script:ListItems = @($itemsJson | ConvertFrom-Json)
        $script:Summary.ItemsFound = $script:ListItems.Count
        
        if ($script:ListItems.Count -eq 0) {
            Write-Host "No items found with non-null $FieldName field. Exiting." -ForegroundColor Yellow
            Stop-Transcript
            return
        }
        
        Write-Host "Found $($script:ListItems.Count) item(s) to process" -ForegroundColor Green
        Write-Host ""
    }
    catch {
        throw "Error retrieving list items: $($_.Exception.Message)"
    }
}

process {
    $index = 0
    $totalCount = $script:ListItems.Count
    
    foreach ($item in $script:ListItems) {
        $index++
        $itemId = $item.Id
        $retryCount = 0
        $success = $false
        
        Write-Verbose "[$index/$totalCount] Processing item ID: $itemId"
        
        do {
            try {
                if ($PSCmdlet.ShouldProcess("Item $itemId", "Clear field '$FieldName'")) {
                    $retryCount++
                    
                    if ($retryCount -gt 1) {
                        Write-Verbose "  Attempt $retryCount/$MaxRetries for item $itemId"
                    }
                    
                    $updateResult = m365 spo listitem set --webUrl $SiteUrl --listTitle $ListName --id $itemId --$FieldName "" --systemUpdate --output json 2>&1
                    
                    if ($LASTEXITCODE -eq 0) {
                        $success = $true
                        $script:Summary.Updated++
                        
                        $reportEntry = [PSCustomObject]@{
                            ItemId = $itemId
                            Status = "Success"
                            Attempts = $retryCount
                            Error = ""
                        }
                        $null = $script:ReportCollection.Add($reportEntry)
                        
                        Write-Verbose "  ✓ Successfully cleared $FieldName for item $itemId (attempt $retryCount)"
                    }
                    else {
                        throw "CLI command failed with exit code $LASTEXITCODE: $updateResult"
                    }
                }
                else {
                    $success = $true
                    Write-Verbose "  [WhatIf] Would clear $FieldName for item $itemId"
                }
            }
            catch {
                $errorMessage = $_.Exception.Message
                
                if ($retryCount -ge $MaxRetries) {
                    Write-Warning "Failed to update item $itemId after $MaxRetries attempts: $errorMessage"
                    $script:Summary.Failures++
                    
                    $reportEntry = [PSCustomObject]@{
                        ItemId = $itemId
                        Status = "Failed"
                        Attempts = $retryCount
                        Error = $errorMessage
                    }
                    $null = $script:ReportCollection.Add($reportEntry)
                    $success = $true
                }
                else {
                    Write-Verbose "  Retry $retryCount/$MaxRetries for item $itemId after $RetryDelaySeconds seconds..."
                    Write-Verbose "  Error: $errorMessage"
                    Start-Sleep -Seconds $RetryDelaySeconds
                }
            }
        } while (-not $success)
        
        if ($index % 100 -eq 0) {
            Write-Host "Progress: $index/$totalCount items processed ($(($index/$totalCount*100).ToString('0.0'))%)" -ForegroundColor Cyan
        }
    }
}

end {
    $endTime = Get-Date
    $duration = $endTime - $startTime
    
    Write-Host ""
    Write-Host "=== Update Summary ===" -ForegroundColor Cyan
    Write-Host "Items found: $($script:Summary.ItemsFound)" -ForegroundColor White
    Write-Host "Successfully updated: $($script:Summary.Updated)" -ForegroundColor $(if ($script:Summary.Updated -gt 0) { 'Green' } else { 'White' })
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failed: $($script:Summary.Failures)" -ForegroundColor Red
    }
    else {
        Write-Host "Failed: 0" -ForegroundColor Green
    }
    
    Write-Host "Duration: $($duration.Hours)h $($duration.Minutes)m $($duration.Seconds)s" -ForegroundColor White
    
    if ($script:ReportCollection.Count -gt 0 -and -not $WhatIfPreference) {
        $csvPath = Join-Path $OutputPath "ListUpdate_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host ""
        Write-Host "Report exported to: $csvPath" -ForegroundColor Green
    }
    
    Write-Host ""
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Green
    Stop-Transcript
}

# .\Update-LargeListField.ps1 -SiteUrl "https://contoso.sharepoint.com/teams/app-test" -ListName "List1" -FieldName "Type"

# .\Update-LargeListField.ps1 -SiteUrl "https://contoso.sharepoint.com/teams/app-test" -ListName "List1" -FieldName "Type" -WhatIf

# .\Update-LargeListField.ps1 -SiteUrl "https://contoso.sharepoint.com/teams/app-test" -ListName "List1" -FieldName "Type" -Verbose

# .\Update-LargeListField.ps1 -SiteUrl "https://contoso.sharepoint.com/teams/app-test" -ListName "List1" -FieldName "Type" -MaxRetries 5 -RetryDelaySeconds 60 -OutputPath "C:\Reports"

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

## Source Credit

Sample first appeared on [Optimising Large List Updates with PnP Batch: Handling Throttling and Enhancing Efficiency](https://reshmeeauckloo.com/posts/pnpbatch-update-biglist-sharepoint/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011)|
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]

<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-largelist-pnpbatch-with-retry" aria-hidden="true" />

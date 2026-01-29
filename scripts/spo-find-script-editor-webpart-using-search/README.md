

# Find every page that contains a Modern Script Editor web part

> [!Note]
> This is a submission helper template please find the [contributor guidance](/docfx/contribute.md) to help you write this scenario.

## Summary

Since the Modern Script Editor web part is impacted by the automatic disabling of Custom Scripting every 24 hours, it is a good idea to find all pages that contain this web part. This script searches for all pages containing the Modern Script Editor web part using SharePoint search and exports the results to CSV.

![Example Screenshot](assets/example.png)


# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
  [Parameter(Mandatory = $false, HelpMessage = "Path to save the CSV report. Defaults to current directory")]
  [string]$OutputPath = (Get-Location).Path
)

begin {
  $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
  $transcriptPath = Join-Path $OutputPath "ScriptEditorWebPart_Transcript_$timestamp.log"
  Start-Transcript -Path $transcriptPath | Out-Null

  Write-Host "Starting Script Editor Web Part search..." -ForegroundColor Cyan

  if ($PSBoundParameters.ContainsKey('OutputPath')) {
    if (-not (Test-Path $OutputPath)) {
      Write-Host "Creating output directory: $OutputPath" -ForegroundColor Yellow
      New-Item -ItemType Directory -Path $OutputPath -Force | Out-Null
    }
  }

  Write-Host "Authenticating to Microsoft 365..." -ForegroundColor Cyan
  m365 login --ensure
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to authenticate to Microsoft 365. Exiting."
  }
  Write-Host "Successfully authenticated." -ForegroundColor Green

  $script:WebPartId = "3a328f0a-99c4-4b28-95ab-fe0847f657a3"
  $script:Results = [System.Collections.ArrayList]@()
  $script:TotalPagesFound = 0
}

process {
 try {
   Write-Host "Searching for pages with Script Editor web part (ID: $script:WebPartId)..." -ForegroundColor Cyan

   $query = 'SPFxExtensionJson:"' + $script:WebPartId + '"'
   Write-Verbose "KQL Query: $query"

    $searchResultsJson = m365 spo search --queryText $query --selectProperties "Path,FileName" --allResults --output json
    if ($LASTEXITCODE -ne 0) {
      Write-Warning "Search query failed. Check if you have appropriate permissions."
      return
    }

    $searchResults = @($searchResultsJson | ConvertFrom-Json)
    $script:TotalPagesFound = $searchResults.Count

    Write-Host "Found $($script:TotalPagesFound) page(s) with Script Editor web part." -ForegroundColor $(if ($script:TotalPagesFound -gt 0) { "Yellow" } else { "Green" })

    if ($script:TotalPagesFound -gt 0) {
      foreach ($result in $searchResults) {
        $script:Results.Add([PSCustomObject]@{
            Path          = $result.Path
            FileName      = $result.FileName
            WebPartId     = $script:WebPartId
            ScanTimestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
          }) | Out-Null
      }
    }
  }
  catch {
    Write-Warning "Error during search execution: $($_.Exception.Message)"
  }
}

end {
  if ($script:Results.Count -gt 0) {
    $csvPath = Join-Path $OutputPath "ScriptEditorWebParts_$timestamp.csv"
    if ($PSCmdlet.ShouldProcess($csvPath, "Export search results to CSV")) {
      $script:Results | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
      Write-Host "Results exported to: $csvPath" -ForegroundColor Green
    }
  }
  else {
    Write-Host "No pages with Script Editor web part found. No CSV file created." -ForegroundColor Green
  }

  Write-Host "`nSummary:" -ForegroundColor Cyan
  Write-Host "  Pages Found: $script:TotalPagesFound" -ForegroundColor $(if ($script:TotalPagesFound -gt 0) { "Yellow" } else { "Green" })

  Stop-Transcript | Out-Null
  Write-Host "`nTranscript saved to: $transcriptPath" -ForegroundColor Gray
}

# Example 1: Run search with default output path (current directory)
# .\Find-ScriptEditorWebParts.ps1

# Example 2: Run search with custom output path
# .\Find-ScriptEditorWebParts.ps1 -OutputPath "C:\Reports"

# Example 3: Preview what would happen without making changes (WhatIf mode)
# .\Find-ScriptEditorWebParts.ps1 -WhatIf

# Example 4: Run with verbose output to see detailed search query
# .\Find-ScriptEditorWebParts.ps1 -Verbose

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

#locate all page containing a Script Editor Web part

$url = "https://contoso.sharepoint.com/sites/somesite"
#login in a way that allows you to search all sites. I usually use -ManagedIdentity in Azure Automation/Function
$conn = Connect-PnPOnline -Url $url -Interactive  -ReturnConnection -WarningAction Ignore

$IdForScriptEditorWebPart = "3a328f0a-99c4-4b28-95ab-fe0847f657a3"
$query = 'SPFxExtensionJson:"'+$IdForScriptEditorWebPart+'"'
$result = Invoke-PnPSearchQuery -Query $query -Connection $conn -All -SelectProperties "Path", "FileName"
$result.ResultRows.Count
#export to csv
$data = @()
foreach($row in $result.ResultRows)
{
    $item = New-Object PSObject
    $item | Add-Member -MemberType NoteProperty -Name "Path" -Value $row.Path
    $item | Add-Member -MemberType NoteProperty -Name "FileName" -Value $row.FileName
    $data += $item
}
$data | Export-Csv -Path "C:\temp\searchresult.csv" -NoTypeInformation 


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Kasper Larsen |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-find-script-editor-webpart-using-search" aria-hidden="true" />



# Run A Search Query And Export To CSV

## Summary

Perform a search query (such as "Show me all News Posts in this tenant") and export the results to CSV.

This script is designed as a starter for you to expand by modifying the query used, and adding whichever managed properties you want to appear in the CSV file, in the order you want.

Any content you can retrieve through search you can use in this script, so as long as you can build the query for it.

The key to this script is the `Submit-PnPSearchQuery` cmdlet, which you can also modify in this script, for example to set the Result Source. See more information on the usage of this cmdlet [here](https://pnp.github.io/powershell/cmdlets/Submit-PnPSearchQuery.html).

![Example Screenshot](assets/example.png)

## Instructions

- Open your favourite text/script editor and copy/paste the script template below 
- Modify the search query to your requirements, add the desired Managed Properties to the Select Properties list
- Also update the `PSCustomObject` with the properties that you require in the resulting CSV file
- Open a PowerShell terminal
- Connect to your SharePoint tenancy using PnP PowerShell
- Run the script
- Retrieve the generated CSV file

# [PnP PowerShell](#tab/pnpps)

``` powershell
$itemsToSave = @()

$query = "PromotedState:2"
$properties = "Title,Path,Author"

$search = Submit-PnPSearchQuery -Query $query -SelectProperties $properties -All

foreach ($row in $search.ResultRows) {


  $data = [PSCustomObject]@{
    "Title"      = $row["Title"]
    "Author"     = $row["Author"]
    "Path"       = $row["Path"]
  }

  $itemsToSave += $data
}

$itemsToSave | Export-Csv -Path "SearchResults.csv" -NoTypeInformation
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]


# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory = $true, HelpMessage = 'SharePoint search query text (KQL format).')]
    [string]$QueryText,

    [Parameter(Mandatory = $false, HelpMessage = 'Comma-separated managed properties to include in the results.')]
    [string]$SelectProperties = 'Title,Path,Author'
)

# Log in to Microsoft 365
Write-Host "Ensuring connection to Microsoft 365" -ForegroundColor Yellow
$loginOutput = m365 login --ensure 2>&1
if ($LASTEXITCODE -ne 0) {
    throw "Failed to authenticate. CLI output: $loginOutput"
}

$searchOutput = m365 spo search --queryText $QueryText --selectProperties $SelectProperties --allResults --output csv 2>&1
if ($LASTEXITCODE -ne 0) {
    throw "Search query failed. CLI output: $searchOutput"
}

$searchOutput | Out-File -FilePath "SearchResults.csv"
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***


## Contributors

| Author(s) |
|-----------|
| James Love |
| Smita Nachan |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-search-export-to-csv" aria-hidden="true" />

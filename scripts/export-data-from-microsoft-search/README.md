

# Export data from MS Search


## Summary

When working with data imported into the Microsoft Search service using Graph Connectors, it can be useful to export the data to a format of your choice for further analysis or to import it into another system. 

![Example Screenshot](assets/example.png)

You will have to connect just as usual, then you must specify the entity type you want , the fields (if you don't know the names of the fields, you can look in the Search And Intelligence Admin Center) or export the schema using Get-PnPSearchExternalSchema
Finally you have to provide the name of the external data source (you can get this from the Search And Intelligence Admin Center)


# [PnP PowerShell](#tab/pnpps)

```powershell

$clientId = "aaaaaa-11111-222222-bbbbb-44444444"
$portalConn = Connect-PnPOnline -Url "https://contoso.sharepoint.com" -Interactive -ClientId $clientId -ReturnConnection

$content = @{
    "Requests" = @(
        @{
            "entityTypes" = @(
                "externalItem"
            )
            "query" = @{
                "queryString" = "*"
            }
            "contentSources" = @(
                "/external/connections/AzureSqlConnector3"
            )
            "fields"= @(
                "CustomerName",
                "CustomerArea",
                "LocationID",
                "LocationName",
                "Responsible"
            )
        }
    )
}

#converting the content to json in order to use it in the Graph Explorer, which is a great way to test the queries
$json = $content | ConvertTo-Json -Depth 10
$res = Invoke-PnPGraphMethod -Url "/v1.0/search/query" -Method Post  -Content $content -Connection $portalConn


$hits = $res.value.hitsContainers.hits 
foreach($hit in $hits)
{
    $hit.resource.properties
    #do something with the data, like exporting it to a csv file
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory, HelpMessage = "Name of the Microsoft Search external connection.")]
    [string]$ConnectionName,

    [Parameter(Mandatory, HelpMessage = "Entity type to filter on (for example externalItem).")]
    [ValidateNotNullOrEmpty()]
    [string]$EntityType,

    [Parameter(HelpMessage = "Maximum number of items to export (default 100).")]
    [ValidateRange(1, 10000)]
    [int]$Top = 100,

    [Parameter(HelpMessage = "Comma separated list of additional fields to include in the export.")]
    [string[]]$Fields,

    [Parameter(HelpMessage = "Local path of the CSV file that will contain the exported data.")]
    [string]$OutputPath = (Join-Path -Path (Get-Location) -ChildPath 'search-export.csv')
)

begin {
    $script:Summary = [ordered]@{
        Connection   = $ConnectionName
        EntityType   = $EntityType
        ItemsFetched = 0
        Failures     = 0
    }

    m365 login --ensure

    if ($PSCmdlet.ShouldProcess($OutputPath, 'Initialize output file')) {
        if (Test-Path -Path $OutputPath) {
            Remove-Item -Path $OutputPath -Force
        }
    }

    Write-Host "Exporting search data from '$ConnectionName' (entity type '$EntityType')" -ForegroundColor Cyan
}

process {
    $arguments = [System.Collections.Generic.List[string]]::new()
    $arguments.AddRange(@('search','external','item','list','--connection', $ConnectionName,'--entityType', $EntityType,'--top', "$Top",'--output','json'))

    if ($Fields) {
        $arguments.Add('--fields')
        $arguments.Add($Fields -join ',')
    }

    $result = (& m365 $arguments.ToArray()) 2>&1

    if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve search data. CLI: $result"
        $Summary.Failures++
        return
    }

    $items = if ([string]::IsNullOrWhiteSpace($result)) { @() } else { $result | ConvertFrom-Json }

    if (-not $items) {
        Write-Host "No items returned for the specified query."
        return
    }

    $Summary.ItemsFetched += $items.Count

    if ($PSCmdlet.ShouldProcess($OutputPath, "Append $($items.Count) item(s)")) {
        $items | ConvertTo-Csv -NoTypeInformation | Out-File -FilePath $OutputPath -Encoding UTF8 -Append
    }
}

end {
    Write-Host "Export complete." -ForegroundColor Green
    Write-Host ("  Connection   : {0}" -f $Summary.Connection)
    Write-Host ("  Entity Type  : {0}" -f $Summary.EntityType)
    Write-Host ("  Items fetched: {0}" -f $Summary.ItemsFetched)
    Write-Host ("  Failures     : {0}" -f $Summary.Failures)
    Write-Host ("  Output file  : {0}" -f $OutputPath)
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/export-data-from-microsoft-search" aria-hidden="true" />

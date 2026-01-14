

# Import CSV values to an existing SharePoint List

## Summary

Main idea here is import content from a csv to a existing list.  

Usually for that to happen,  we need to explicit enumerate each column of the list and in the csv file.    

With this sample you dont need to do it anymore as long you follow the bellow rule :  
>  
> CSV columns must have the same name as the list columns name.  
  
Excelsior, hum? :P  

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding()]
param (
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)')]
    [string]$WebUrl,

    [Parameter(Mandatory, HelpMessage = "Title of the SharePoint list")]
    [string]$ListTitle,

    [Parameter(Mandatory, HelpMessage = "Path to the CSV file containing list items")]
    [ValidateScript({ Test-Path $_ -PathType Leaf })]
    [string]$CsvFilePath,

    [Parameter(HelpMessage = "Path where transcript log will be saved")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Verbose "Ensuring login to Microsoft 365..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365"
    }

    Write-Verbose "Validating CSV file: $CsvFilePath"
    $csvData = Import-Csv -Path $CsvFilePath
    $totalRows = $csvData.Count
    Write-Host "Found $totalRows rows in CSV file" -ForegroundColor Cyan

    $script:Summary = @{
        TotalRows = $totalRows
        Success   = $false
        Failures  = 0
    }

    $transcriptPath = Join-Path $OutputPath "spo-import-csv_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    Start-Transcript -Path $transcriptPath
    Write-Host "Transcript started: $transcriptPath" -ForegroundColor Cyan
}

process {
    Write-Host "Importing $($script:Summary.TotalRows) items to list '$ListTitle'..." -ForegroundColor Cyan

    try {
        $output = m365 spo listitem batch add --webUrl $WebUrl --listTitle $ListTitle --filePath $CsvFilePath 2>&1

        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Batch add operation failed: $output"
            $script:Summary.Failures++
        }
        else {
            $script:Summary.Success = $true
        }
    }
    catch {
        Write-Warning "Exception during batch add: $($_.Exception.Message)"
        $script:Summary.Failures++
    }
}

end {
    Stop-Transcript

    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Import Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Total Rows in CSV: $($script:Summary.TotalRows)"

    if ($script:Summary.Success) {
        Write-Host "Status: SUCCESS" -ForegroundColor Green
        Write-Host "All items were imported successfully!" -ForegroundColor Green
    }
    else {
        Write-Host "Status: FAILED" -ForegroundColor Red
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
        Write-Host "Review the transcript log for details: $transcriptPath" -ForegroundColor Yellow
    }

    Write-Host "========================================" -ForegroundColor Cyan
}

# Example 1: Import CSV to list using site URL and list title
# .\your-script.ps1 -WebUrl "https://contoso.sharepoint.com/sites/project-x" -ListTitle "Demo List" -CsvFilePath "C:\Data\items.csv"

# Example 2: Import with verbose output
# .\your-script.ps1 -WebUrl "https://contoso.sharepoint.com/sites/project-x" -ListTitle "Demo List" -CsvFilePath "C:\Data\items.csv" -Verbose

# Example 3: Import with custom transcript location
# .\your-script.ps1 -WebUrl "https://contoso.sharepoint.com/sites/project-x" -ListTitle "Demo List" -CsvFilePath "C:\Data\items.csv" -OutputPath "C:\Logs"

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell

[CmdletBinding()]
param (
    [Parameter(Mandatory = $true)]
    [string]$Url,
    [Parameter(Mandatory = $true)]
    [string]$ListName,
    [Parameter(Mandatory = $true)]
    [string]$CsvFile
)
begin {
    Import-Module PnP.PowerShell
    Write-Output "Connecting to $Url"
    Connect-PnPOnline -Url $Url -Interactive
}
process {
    
    
    ## Powershell filter , it converts an array in a hashtable
    filter ArrayToHash {
        begin {
            $hash = @{} 
        }
        process { 
            $obj = $_ | Get-Member | Where-Object { $_.MemberType -eq 'NoteProperty' } | Select-object name
            foreach ($o in $obj) {
                $name = $o.Name
                $hash[$name] = $_."$name"
            }
     
        }
        end { return $hash }
    }
    Write-Output " Collect CSV data from $CsvFile"
    $rows = Import-Csv $CsvFile
    $totalRows = $rows.Length
    Write-Output " $CsvFile has $totalRows rows"
 
    Write-Output " Items will be added using batch mode"
    Write-Output " Initiate batch" 
    $batch = New-PnPBatch
    $ct=0
    $rows.ForEach({
            # convert hast
            $values = $_ | ArrayToHash
            Add-PnPListItem -List $ListName -Values $values -Batch  $batch
            Write-Output "  Item added ($ct/$totalRows)"  
            $ct++
        })
    Write-Output " Invoke batch" 
    Invoke-PnPBatch -Batch $batch
    Write-Output " Batch invoked"
}
end{
    Write-Output " Disconnecting"
    Disconnect-PnPOnline
    Write-Output "Disconnected from $Url"
    Write-Output "All done!"
}
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| Rodrigo Pinto |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-import-csv-data-to-existing-sharepoint-list" aria-hidden="true" />

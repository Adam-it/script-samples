

# Bulk remove retention labels from files in a SharePoint Library

## Summary

Bulk remove all retention labels from all files in a library that are labelled with the retention label. These files will now no longer be tagged with a retention label.

This sample provides two implementations:
- **CLI for Microsoft 365**: Processes items individually with detailed progress tracking and error handling
- **PnP PowerShell**: Uses the bulk API for faster processing of large batches (up to 200 items per call)

I had a requirement to complete a migration again but first needed to remove the labels from the files before I could complete the re-migration. It was slow removing the labels one by one using PowerShell and was quicker using the UI by bulk selecting 200 files and then removing the label in the details pain. I then looked using the developer tools to see how this took place behind the scenes and saw it uses the endpoint **/_api/SP.CompliancePolicy.SPPolicyStoreProxy.ApplyLabelOnBulkItems()**

This script
- Finds all the files in a library tagged with a specified label(s) and obtains their ID
- Splits the list of IDs into batches of 200 (the max supported per call to endpoint)
- Creates a JSON payload formulated with the library details and the list item IDs (max 200) of labelled files.
- Send this JSON payload to /_api/SP.CompliancePolicy.SPPolicyStoreProxy.ApplyLabelOnBulkItems() to remove all labels from the files.
- Repeats in batches of 200 until all the labels are removed from named labelled files in a library.

See below for how the above can be done in the UI but use this script instead and save lots of button clicking :)
![Example Screenshot](assets/example.png)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL")]
    [string]$SiteUrl,
    
    [Parameter(Mandatory = $true, HelpMessage = "Name(s) of retention labels to remove")]
    [string[]]$LabelsToRemove,
    
    [Parameter(Mandatory = $true, HelpMessage = "Name(s) of document libraries to process")]  
    [string[]]$Libraries,
    
    [Parameter(HelpMessage = "Export results to CSV file")]
    [switch]$ExportToCsv,
    
    [Parameter(HelpMessage = "Path for CSV export")]
    [string]$CsvPath = "RemovedLabels_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
)

begin {
    # Ensure user is logged in
    Write-Verbose "Ensuring user is logged into CLI for Microsoft 365..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to CLI for Microsoft 365. Please run 'm365 login' first."
    }
    
    # Validate site URL
    Write-Verbose "Validating site URL: $SiteUrl"
    $siteJson = m365 spo site get --url $SiteUrl --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Cannot access site '$SiteUrl'. Please verify the URL and your permissions. CLI: $siteJson"
    }
    
    # Initialize summary tracking
    $script:Summary = @{
        TotalItemsFound = 0
        LabelsRemoved = 0
        Failures = 0
        ItemsProcessed = [System.Collections.ArrayList]@()
    }
    
    # Initialize throttling counter
    $script:ThrottleCounter = 0
}

process {
    Write-Host "Processing $($Libraries.Count) libraries for $($LabelsToRemove.Count) retention labels..." -ForegroundColor Cyan
    
    foreach ($libraryName in $Libraries) {
        Write-Host "`nProcessing library: $libraryName" -ForegroundColor Yellow
        
        # Get library details with specific filter
        Write-Verbose "Retrieving library information..."
        $libraryJson = m365 spo list list --webUrl $SiteUrl --query "[?Title == '$libraryName']" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to retrieve library '$libraryName'. CLI: $libraryJson"
            $script:Summary.Failures++
            continue
        }
        
        $library = $libraryJson | ConvertFrom-Json
        if (-not $library -or $library.Count -eq 0) {
            Write-Warning "Library '$libraryName' not found."
            $script:Summary.Failures++
            continue
        }
        $library = $library[0]
        
        foreach ($label in $LabelsToRemove) {
            Write-Host "  Searching for items with label: $label" -ForegroundColor Gray
            
            # Get all items with this retention label using server-side filtering
            Write-Verbose "Retrieving items with retention label '$label'..."
            $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listId $library.Id --filter "ComplianceTag eq '$label'" --fields "Id,FileRef,ComplianceTag" --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve items with label '$label' from library '$libraryName'. CLI: $itemsJson"
                $script:Summary.Failures++
                continue
            }
            
            $items = @($itemsJson | ConvertFrom-Json)
            if ($items.Count -eq 0) {
                Write-Host "    No items found with label '$label'" -ForegroundColor Gray
                continue
            }
            
            Write-Host "    Found $($items.Count) items with label '$label'" -ForegroundColor Green
            $script:Summary.TotalItemsFound += $items.Count
            
            # Process items with progress bar
            $itemCount = 0
            $totalItems = $items.Count
            
            foreach ($item in $items) {
                $itemCount++
                $percentComplete = ($itemCount / $totalItems) * 100
                Write-Progress -Activity "Removing label '$label' from library '$libraryName'" -Status "Processing item $itemCount of $totalItems" -PercentComplete $percentComplete -Id 1
                
                if ($PSCmdlet.ShouldProcess($item.FileRef, "Remove retention label '$label'")) {
                    try {
                        # Remove retention label from item
                        Write-Verbose "Removing label from item ID $($item.Id): $($item.FileRef)"
                        $removeResult = m365 spo listitem retentionlabel remove --webUrl $SiteUrl --listId $library.Id --listItemId $item.Id --force 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to remove label from item '$($item.FileRef)'. CLI: $removeResult"
                            $script:Summary.Failures++
                            continue
                        }
                        
                        $script:Summary.LabelsRemoved++
                        
                        # Track for CSV export
                        [void]$script:Summary.ItemsProcessed.Add([PSCustomObject]@{
                            Library = $libraryName
                            ItemPath = $item.FileRef
                            RemovedLabel = $label
                            Timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                        })
                        
                        Write-Verbose "Successfully removed label from: $($item.FileRef)"
                        
                        # Implement throttling to avoid API limits (every 50 items)
                        $script:ThrottleCounter++
                        if ($script:ThrottleCounter % 50 -eq 0) {
                            Write-Verbose "Throttling: Pausing for 500ms after processing 50 items..."
                            Start-Sleep -Milliseconds 500
                        }
                    }
                    catch {
                        Write-Warning "Error removing label from item '$($item.FileRef)': $_"
                        $script:Summary.Failures++
                    }
                }
            }
            
            Write-Progress -Activity "Removing label '$label' from library '$libraryName'" -Id 1 -Completed
        }
    }
}

end {
    # Display summary
    Write-Host "`n" -NoNewline
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "       Retention Label Removal Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "  Total Items Found : $($Summary.TotalItemsFound)" -ForegroundColor White
    Write-Host "  Labels Removed    : $($Summary.LabelsRemoved)" -ForegroundColor Green
    
    if ($Summary.Failures -gt 0) {
        Write-Host "  Failures          : $($Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "  Failures          : 0" -ForegroundColor Green
    }
    
    Write-Host "========================================" -ForegroundColor Cyan
    
    # Export to CSV if requested
    if ($ExportToCsv -and $Summary.ItemsProcessed.Count -gt 0) {
        try {
            $Summary.ItemsProcessed | Export-Csv -Path $CsvPath -NoTypeInformation
            Write-Host "`nResults exported to: $CsvPath" -ForegroundColor Green
        }
        catch {
            Write-Warning "Failed to export results to CSV: $_"
        }
    }
    elseif ($ExportToCsv) {
        Write-Host "`nNo items processed to export." -ForegroundColor Yellow
    }
    
    # Display sample results if not exported
    if (-not $ExportToCsv -and $Summary.ItemsProcessed.Count -gt 0) {
        Write-Host "`nSample of processed items:" -ForegroundColor Cyan
        $Summary.ItemsProcessed | Select-Object -First 10 | Format-Table -AutoSize
        if ($Summary.ItemsProcessed.Count -gt 10) {
            Write-Host "... and $($Summary.ItemsProcessed.Count - 10) more items" -ForegroundColor Gray
        }
    }
}

# Usage Examples:
# Basic usage - remove specific labels from specific libraries
# .\Remove-RetentionLabels.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/EntitySite" -LabelsToRemove "Entity Document","Entity Record" -Libraries "Documents","Reports"

# With CSV export
# .\Remove-RetentionLabels.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/EntitySite" -LabelsToRemove "Entity Document" -Libraries "Documents" -ExportToCsv -CsvPath "C:\Reports\removed_labels.csv"

# WhatIf mode - preview what would be removed
# .\Remove-RetentionLabels.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/EntitySite" -LabelsToRemove "Entity Document" -Libraries "Documents" -WhatIf

# With verbose output for troubleshooting
# .\Remove-RetentionLabels.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/EntitySite" -LabelsToRemove "Entity Document" -Libraries "Documents" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
#----------------------------------------------------------[Local variables to update]----------------------------------------------------------
$labelsToRemove =  "Entity Document","Entity Record" #Enter name of labels to remove
$siteURL = "https://contoso.sharepoint.com/sites/EntitySite" # Enter site url
$libraries = "Documents","Reports" # Enter Document Library Names Here
#---------------------------------------------------------[Initialisation]--------------------------------------------------------
Clear-Host
#-----------------------------------------------------------[Execution]-----------------------------------------------------------


function Split-Collection {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [object[]] $InputObject,

        [Parameter(Position = 0)]
        [ValidateRange(1, [int]::MaxValue)]
        [int] $ChunkSize = 5
    )

    begin {
        $list = [System.Collections.Generic.List[object]]::new()
    }
    process {
        foreach($item in $InputObject) {
            $list.Add($item)
            if($list.Count -eq $ChunkSize) {
                $PSCmdlet.WriteObject($list.ToArray())
                $list.Clear()
            }
        }
    }
    end {
        if($list.Count) {
            $PSCmdlet.WriteObject($list.ToArray())
        }
    }
}

Write-Host "Connecting to site: $($siteURL)" -foregroundcolor Green

try
{
    Connect-PnPOnline -url $siteURL -Interactive
    $ctx = Get-PnPContext
}
catch
{
    write-host  "You don't have permission to site - $($siteURL)" -foregroundcolor Red
    write-host  "Error: $($_.Exception.Message)" -foregroundcolor Red
    exit
}


foreach($label in $labelsToRemove)
{
    foreach($lib in $libraries)
    {
        try
        {
            Write-Host "Processing $lib library in $siteURL"
            $DocumentsLib = Get-PnPList -Identity $lib

            #Gettings All Items with Label - workaround if more than 5000 items as CAML can then not be used.
            if($DocumentsLib.ItemCount -gt 4999)
            {
                $items = Get-PnPListItem -List $DocumentsLib -PageSize 5000 | Where-Object { $_.FieldValues._ComplianceTag -eq $label} | Select-Object Id
            }
            else 
            {
                $CAML = "<View><Query><Where><Eq><FieldRef Name='_ComplianceTag' /><Value Type='Text'>$label</Value></Eq></Where></Query></View>"
                $items = Get-PnPListItem -List $DocumentsLib -PageSize 5000 -Query $caml | Select-Object Id
            }

            if($items.Count -gt 0)
            {
                Write-Host "   $($items.Count) items with $label label found for $($lib.Title)" -ForegroundColor Magenta
                
                $items.Id | Split-Collection 200 | ForEach-Object {

                $JSON ="{
                    ""blockDelete"": false,
                    ""blockEdit"": false,
                    ""complianceTagValue"": """",
                    ""itemIds"": [
                        $($_ -join "","")
                    ],
                    ""listUrl"": ""$($DocumentsLib.RootFolder.ServerRelativeUrl)""
                }"

                Write-Host "     Created JSON Payload to Remove Label: $label from $($_.Count) files" -ForegroundColor Yellow

                Invoke-PnPSPRestMethod -Method Post -Url "$($ctx.Url)/_api/SP.CompliancePolicy.SPPolicyStoreProxy.ApplyLabelOnBulkItems()"  -ContentType "application/json;odata=verbose" -Content $JSON
                }
            }
            else
            {
                Write-Host "   No items with $label label found for $($lib.Title)" -ForegroundColor Green
            }
        }
        catch
        {
            Write-Host  "Error: $($_.Exception.Message)" -ForegroundColor Red
        }
    }
}


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| [Leon Armston](https://github.com/LeonArmston) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-bulk-remove-retention-labels" aria-hidden="true" />

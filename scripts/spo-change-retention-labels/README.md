

# M365 Consultant's Script Kit

## Summary

This script is part of the Microsoft 365 Consultant's Script kit. It scans SharePoint sites (or OneDrive URLs) and updates retention labels on list items based on a configurable mapping. Available in both PnP PowerShell and CLI for Microsoft 365 versions.

The script addresses the limitation that certain retention label actions cannot be changed without creating a new label, providing an automated way to bulk-update items across multiple sites.

  ##  Pre-requisites
	PowerShell 7 must be installed

	The following PowerShell modules are needed:
	PnP.PowerShell
	
## Setup
### Prepare the CSV file with SharePoint site URLs:
1. Use the included URLStoScan csv file
2. Place the urls in the csv file 
###	Edit the script:
1.	Open the script file in a text editor (or Visual Studio Code).
2.	Modify the variable $CsvFilePath to specify the path to the CSV file you are using if necessary
#### Customize retention label updates:
1.	Review the if and elseif statements within the script.
2. change the first two "if/else" statements to include the your old and new labels
3. If you have more than two, uncomment and modify the sections related to retention labels that you want to update based on your specific requirements.
For example, if you want to update the "25 years" retention label, uncomment the section and modify it as follows:

elseif ($ExistingRetentionLabel -eq "25 years") {
    Set-PnPListItem -List $List -Identity $Item.Id -Values @{ "_ComplianceTag" = "New 25 Years Label" }
}

![Setup your labels](assets/retentionlabels.png)
 
4.	Save the modified script:

## Running the Script

Now that the scripts are setup, you just need to run them. All these steps are the same, just change the name of the script.
1.	Open PowerShell 7 (as administrator recommended)
2.	Type CD “<whatever the path is where these scripts are>”
3.	Run the Script

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "Path to CSV file containing site URLs (column: SiteUrl)")]
    [ValidateScript({Test-Path $_ -PathType Leaf})]
    [string]$CsvPath,
    
    [Parameter(Mandatory = $true, HelpMessage = "Hashtable mapping old label names to new label names, e.g., @{'Old Label 1' = 'New Label 1'; 'Old Label 2' = 'New Label 2'}")]
    [hashtable]$LabelMapping,
    
    [Parameter(HelpMessage = "Export results to CSV file")]
    [switch]$ExportToCsv,
    
    [Parameter(HelpMessage = "Path for CSV export file")]
    [string]$OutputPath = ".\\RetentionLabelChanges.csv"
)

begin {
    $currentTime = $(Get-Date).ToString("yyyyMMddHHmmss")
    $logFilePath = ".\\log-$currentTime.log"
    Start-Transcript -Path $logFilePath
    
    $script:Summary = @{
        SitesProcessed    = 0
        ListsProcessed    = 0
        ItemsFound        = 0
        LabelsChanged     = 0
        LabelsSkipped     = 0
        Failures          = 0
    }
    
    $script:Results = [System.Collections.ArrayList]::new()
    
    Write-Verbose "Validating CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' manually."
    }
    
    Write-Verbose "Validating CSV file..."
    $sites = Import-Csv -Path $CsvPath
    
    if ($sites.Count -eq 0) {
        throw "CSV file is empty or has no valid rows."
    }
    
    if (-not ($sites[0].PSObject.Properties.Name -contains 'SiteUrl')) {
        throw "CSV must contain 'SiteUrl' column."
    }
    
    Write-Verbose "Loaded $($sites.Count) site(s) from CSV"
    Write-Verbose "Label mappings configured: $($LabelMapping.Count)"
    
    $script:AllSites = $sites
}

process {
    $siteCount = 0
    
    foreach ($site in $script:AllSites) {
        $siteCount++
        $script:Summary.SitesProcessed++
        $siteUrl = $site.SiteUrl.Trim()
        
        Write-Progress -Activity "Processing Sites" -Status "Site: $siteUrl" -PercentComplete (($siteCount / $script:AllSites.Count) * 100) -Id 0
        Write-Verbose "Processing site $siteCount of $($script:AllSites.Count): $siteUrl"
        
        try {
            Write-Verbose "  Retrieving lists and libraries..."
            $listsJson = m365 spo list list --webUrl $siteUrl --filter "Hidden eq false" --output json 2>&1
            
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve lists from site '$siteUrl'. CLI: $listsJson"
                $script:Summary.Failures++
                continue
            }
            
            $lists = @($listsJson | ConvertFrom-Json)
            Write-Verbose "  Found $($lists.Count) list(s)/librar(ies)"
            
            $listCount = 0
            foreach ($list in $lists) {
                $listCount++
                $script:Summary.ListsProcessed++
                
                Write-Progress -Activity "Processing Lists in $siteUrl" -Status "List: $($list.Title)" -PercentComplete (($listCount / $lists.Count) * 100) -Id 1 -ParentId 0
                Write-Verbose "    Processing list: $($list.Title)"
                
                try {
                    $itemsJson = m365 spo listitem list --webUrl $siteUrl --listId $list.Id --fields "Id,Title,_ComplianceTag" --filter "ComplianceTag ne null" --output json 2>&1
                    
                    if ($LASTEXITCODE -ne 0) {
                        Write-Verbose "      No items with retention labels found or error occurred"
                        continue
                    }
                    
                    $items = @($itemsJson | ConvertFrom-Json)
                    
                    if ($items.Count -eq 0) {
                        Write-Verbose "      No items with retention labels"
                        continue
                    }
                    
                    $script:Summary.ItemsFound += $items.Count
                    Write-Verbose "      Found $($items.Count) item(s) with retention labels"
                    
                    foreach ($item in $items) {
                        $currentLabel = $item._ComplianceTag
                        
                        if ($LabelMapping.ContainsKey($currentLabel)) {
                            $newLabel = $LabelMapping[$currentLabel]
                            
                            Write-Verbose "        Item $($item.Id): '$currentLabel' → '$newLabel'"
                            
                            if ($PSCmdlet.ShouldProcess("Item $($item.Id) in $($list.Title)", "Change retention label from '$currentLabel' to '$newLabel'")) {
                                $status = "Success"
                                $errorMsg = ""
                                
                                try {
                                    $setResult = m365 spo listitem retentionlabel ensure --webUrl $siteUrl --listId $list.Id --listItemId $item.Id --name $newLabel 2>&1
                                    
                                    if ($LASTEXITCODE -ne 0) {
                                        Write-Warning "Failed to update retention label for item $($item.Id) in list '$($list.Title)'. CLI: $setResult"
                                        $script:Summary.Failures++
                                        $status = "Failed"
                                        $errorMsg = $setResult
                                    }
                                    else {
                                    $script:Summary.LabelsChanged++
                                    }
                                } catch {
                                    Write-Warning "Unexpected error updating item $($item.Id): $_"
                                    $script:Summary.Failures++
                                    $status = "Failed"
                                    $errorMsg = $_.Exception.Message
                                }
                                
                                if ($ExportToCsv) {
                                    [void]$script:Results.Add([PSCustomObject]@{
                                        SiteUrl       = $siteUrl
                                        ListTitle     = $list.Title
                                        ItemId        = $item.Id
                                        ItemTitle     = $item.Title
                                        OldLabel      = $currentLabel
                                        NewLabel      = $newLabel
                                        Status        = $status
                                        ErrorMessage  = $errorMsg
                                    })
                                }
                            }
                        } else {
                            Write-Verbose "        Item $($item.Id): Label '$currentLabel' not in mapping - skipped"
                            $script:Summary.LabelsSkipped++
                        }
                    }
                } catch {
                    Write-Warning "Unexpected error processing list '$($list.Title)': $_"
                    $script:Summary.Failures++
                    continue
                }
            }
            
            Write-Progress -Activity "Processing Lists in $siteUrl" -Id 1 -Completed
        } catch {
            Write-Warning "Unexpected error processing site '$siteUrl': $_"
            $script:Summary.Failures++
            continue
        }
    }
    
    Write-Progress -Activity "Processing Sites" -Id 0 -Completed
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Retention Label Change Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "  Sites Processed    : $($Summary.SitesProcessed)" -ForegroundColor White
    Write-Host "  Lists Processed    : $($Summary.ListsProcessed)" -ForegroundColor White
    Write-Host "  Items Found        : $($Summary.ItemsFound)" -ForegroundColor White
    Write-Host "  Labels Changed     : $($Summary.LabelsChanged)" -ForegroundColor Green
    Write-Host "  Labels Skipped     : $($Summary.LabelsSkipped)" -ForegroundColor Yellow
    
    if ($Summary.Failures -gt 0) {
        Write-Host "  Failures           : $($Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "  Failures           : $($Summary.Failures)" -ForegroundColor Green
    }
    
    Write-Host "========================================" -ForegroundColor Cyan
    
    if ($ExportToCsv -and $script:Results.Count -gt 0) {
        $script:Results | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
        Write-Host "`n✅ Results exported to: $OutputPath" -ForegroundColor Green
    }
    
    Stop-Transcript
}

# Usage Examples:
# .\\Change-RetentionLabels.ps1 -CsvPath ".\\sites.csv" -LabelMapping @{"Old Label 1" = "New Label 1"; "Old Label 2" = "New Label 2"}
# .\\Change-RetentionLabels.ps1 -CsvPath ".\\sites.csv" -LabelMapping @{"Script Test 1 Old" = "Script Test 1 New"} -ExportToCsv -OutputPath ".\\results.csv"
# .\\Change-RetentionLabels.ps1 -CsvPath ".\\sites.csv" -LabelMapping @{"25 years" = "New 25 Years Label"; "30 years" = "New 30 Years Label"} -WhatIf
# .\\Change-RetentionLabels.ps1 -CsvPath ".\\sites.csv" -LabelMapping @{"Confidential" = "Highly Confidential"} -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

Start-Transcript -Append log.txt
# Import PnP PowerShell module
Import-Module -Name pnp.powershell -DisableNameChecking


# Read SharePoint site URLs from CSV file
$CsvFilePath = "<PATH>\URLStoScan.csv"
$SiteUrls = Import-Csv -Path $CsvFilePath | Select-Object -ExpandProperty SiteUrl

# Define retention label mapping
$RetentionLabelMapping = @{
    "Script Test 1 Old" = "Script Test 1 New"
    "Script Test 2 Old" = "Script Test 2 New"
    #"25 years"          = "New 25 Years Label"
    #"30 years"          = "New 30 Years Label"
    #"35 years"          = "New 35 Years Label"
}

# Function to process a single list item
function Set-RetentionLabel {
    param (
        [Parameter(Mandatory)]
        $List,

        [Parameter(Mandatory)]
        $Item,

        [Parameter(Mandatory)]
        $FieldInternalName,

        [Parameter(Mandatory)]
        $Mapping
    )

    $CurrentLabel = $Item[$FieldInternalName]
    Write-Host "Item ID: $($Item.Id) | Current label: $CurrentLabel" -ForegroundColor Blue

    if ($Mapping.ContainsKey($CurrentLabel)) {
        $NewLabel = $Mapping[$CurrentLabel]
        Set-PnPListItem -List $List -Identity $Item.Id -Label $NewLabel
        Write-Host "Updated label to: $NewLabel" -ForegroundColor Green
    } else {
        Write-Host "No matching retention label found for Item ID $($Item.Id) in List '$($List.Title)'" -ForegroundColor Yellow
    }
}

# Iterate through each SharePoint site
foreach ($SiteUrl in $SiteUrls) {
    Write-Host "Processing SharePoint site: $SiteUrl" -ForegroundColor Cyan

    Connect-PnPOnline -Url $SiteUrl -Interactive

    # Retrieve all lists and libraries
    $Lists = Get-PnPList | Where-Object { $_.BaseType -in @("GenericList","DocumentLibrary") }
    Write-Host "$($Lists.Count) lists/libraries retrieved." -ForegroundColor Green

    foreach ($List in $Lists) {
        Write-Host "Processing list: $($List.Title)" -ForegroundColor Cyan

        # Get retention label field
        $RetentionField = Get-PnPField -List $List -Identity "_ComplianceTag"
        if (-not $RetentionField) {
            Write-Host "Retention label field not found in list $($List.Title)." -ForegroundColor Red
            continue
        }

        # Retrieve all items
        $Items = Get-PnPListItem -List $List
        Write-Host "$($Items.Count) items found in list $($List.Title)." -ForegroundColor Yellow

        foreach ($Item in $Items) {
            Set-RetentionLabel -List $List -Item $Item -FieldInternalName $RetentionField.InternalName -Mapping $RetentionLabelMapping
        }

        Write-Host "Finished processing list $($List.Title)." -ForegroundColor Green
    }

    Write-Host "Finished processing site $SiteUrl." -ForegroundColor Cyan
}

Stop-Transcript

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]



## Contributors

| Author(s) |
|-----------|
| Nick Brattoli|
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-change-retention-labels" aria-hidden="true" />

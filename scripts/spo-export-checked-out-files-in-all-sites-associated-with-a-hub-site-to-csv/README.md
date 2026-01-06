

# Export checked-out files in all sites associated with a hub site to CSV

## Summary

This script will export all Checked-Out files in all SharePoint Online sites associated with a Hub site to a CSV file.

### Notes

- The CLI for Microsoft 365 version uses `m365 spo listitem list` with OData filtering to retrieve checked-out files, which is simpler than CAML queries.
- Uses `--withAssociatedSites` option to get all hub-associated sites in a single command.

### Prerequisites

- The user account that runs the script must have SharePoint Online tenant administrator access.
- Before running the script, edit the script and update the variable values in the Config Variable section, such as SharePoint Tenant Admin URL, Hub Site URL, and the CSV output file path.

### Screenshots
Screen Output

![Screen Output](assets/screen-output.png)

CSV Output

![CSV Output](assets/csv-output.png)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Hub site URL (e.g., https://contoso.sharepoint.com/sites/LegalHub)")]
    [ValidatePattern('^https://')]
    [string]$HubSiteUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Path for CSV output file")]
    [string]$OutputPath
)

begin {
    # Start transcript
    $timestamp = Get-Date -Format "yyyyMMddHHmmss"
    $transcriptPath = "CheckedOutFiles_$timestamp.log"
    Start-Transcript -Path $transcriptPath

    Write-Host "Starting checked-out files export..." -ForegroundColor Cyan

    # Set default output path if not specified
    if (-not $OutputPath) {
        $OutputPath = "CheckedOutFiles_$timestamp.csv"
    } else {
        # Validate OutputPath directory exists
        $outputDir = Split-Path -Path $OutputPath -Parent
        if ($outputDir -and -not (Test-Path -Path $outputDir)) {
            throw "Output directory does not exist: $outputDir"
        }
    }

    # Ensure user is logged in to Microsoft 365
    Write-Host "Ensuring Microsoft 365 connection..." -ForegroundColor Yellow
    m365 login --ensure

    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365. Please run 'm365 login' first."
    }

    Write-Host "Successfully authenticated." -ForegroundColor Green

    # Initialize collection for checked-out files
    $script:CheckedOutFiles = [System.Collections.ArrayList]::new()

    # Initialize counters
    $script:Summary = @{
        SitesProcessed   = 0
        LibrariesScanned = 0
        FilesFound       = 0
        Failures         = 0
    }

    Write-Host "`nRetrieving hub site and associated sites..." -ForegroundColor Yellow

    # Get hub site with associated sites
    $hubSiteJson = m365 spo hubsite get --url $HubSiteUrl --withAssociatedSites --output json 2>&1

    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve hub site. Error: $hubSiteJson"
    }

    $hubSite = $hubSiteJson | ConvertFrom-Json

    if (-not $hubSite.AssociatedSites -or $hubSite.AssociatedSites.Count -eq 0) {
        Write-Host "No associated sites found for hub site: $HubSiteUrl" -ForegroundColor Yellow
        return
    }

    Write-Host "Found $($hubSite.AssociatedSites.Count) associated site(s)." -ForegroundColor Green
}

process {
    # Process each associated site
    foreach ($associatedSite in $hubSite.AssociatedSites) {
        $siteUrl = $associatedSite.SiteUrl

        try {
            Write-Host "`nProcessing site: $siteUrl" -ForegroundColor Cyan
            $script:Summary.SitesProcessed++

            # Get all document libraries (BaseType 1 = DocumentLibrary)
            Write-Verbose "Retrieving document libraries from $siteUrl..."
            $listsJson = m365 spo list list --webUrl $siteUrl --filter "BaseType eq 1 and Hidden eq false" --output json 2>&1

            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve lists from $siteUrl. Error: $listsJson"
                $script:Summary.Failures++
                continue
            }

            $libraries = @($listsJson | ConvertFrom-Json)

            if ($libraries.Count -eq 0) {
                Write-Verbose "No document libraries found in $siteUrl"
                continue
            }

            Write-Host "  Found $($libraries.Count) document librar(y/ies)" -ForegroundColor Gray

            # Process each document library
            foreach ($library in $libraries) {
                try {
                    Write-Verbose "  Scanning library: $($library.Title)..."
                    $script:Summary.LibrariesScanned++

                    # Get checked-out files using listitem list with OData filter
                    $checkedOutItemsJson = m365 spo listitem list --webUrl $siteUrl --listId $library.Id --fields "FileLeafRef,FileDirRef,File_x0020_Size,Modified,CheckoutUser/Title" --filter "CheckoutUser ne null" --output json 2>&1

                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "    Failed to retrieve checked-out files from library '$($library.Title)'. Error: $checkedOutItemsJson"
                        $script:Summary.Failures++
                        continue
                    }

                    $checkedOutFiles = @($checkedOutItemsJson | ConvertFrom-Json)

                    if ($checkedOutFiles.Count -eq 0) {
                        Write-Verbose "    No checked-out files in library: $($library.Title)"
                        continue
                    }

                    Write-Host "    Found $($checkedOutFiles.Count) checked-out file(s) in '$($library.Title)'" -ForegroundColor Yellow

                    # Add each checked-out file to collection
                    foreach ($file in $checkedOutFiles) {
                        $checkedOutUser = if ($file.CheckoutUser) { $file.CheckoutUser.Title } else { "Unknown" }
                        $fileSizeMB = if ($file.File_x0020_Size) { [Math]::Round(($file.File_x0020_Size / 1MB), 2) } else { 0 }

                        $null = $script:CheckedOutFiles.Add([PSCustomObject]@{
                                SiteUrl         = $siteUrl
                                SiteTitle       = $associatedSite.Title
                                LibraryName     = $library.Title
                                CheckedOutTo    = $checkedOutUser
                                CheckedOutSince = $file.Modified
                                FileSizeMB      = $fileSizeMB
                                FileName        = $file.FileLeafRef
                                FileURL         = $file.FileDirRef
                            })

                        $script:Summary.FilesFound++
                    }
                }
                catch {
                    Write-Warning "    Error processing library '$($library.Title)': $_"
                    $script:Summary.Failures++
                    continue
                }
            }
        }
        catch {
            Write-Warning "Error processing site '$siteUrl': $_"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    # Export results to CSV
    if ($script:CheckedOutFiles.Count -gt 0) {
        Write-Host "`nExporting results to CSV: $OutputPath" -ForegroundColor Cyan
        $script:CheckedOutFiles | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
        Write-Host "Export completed successfully!" -ForegroundColor Green
    }
    else {
        Write-Host "`nNo checked-out files found." -ForegroundColor Yellow
    }

    # Display summary
    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Sites processed:      $($script:Summary.SitesProcessed)" -ForegroundColor White
    Write-Host "Libraries scanned:    $($script:Summary.LibrariesScanned)" -ForegroundColor White
    Write-Host "Checked-out files:    $($script:Summary.FilesFound)" -ForegroundColor $(if ($script:Summary.FilesFound -gt 0) { 'Green' } else { 'Gray' })
    Write-Host "Failures:             $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "Output file:          $OutputPath" -ForegroundColor White
    Write-Host "============================`n" -ForegroundColor Cyan

    Stop-Transcript
}

# Example 1: Export checked-out files from hub site to default CSV file
# .\\Export-CheckedOutFiles.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/LegalHub"

# Example 2: Export with custom output path
# .\\Export-CheckedOutFiles.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/LegalHub" -OutputPath "C:\\Reports\\CheckedOut.csv"

# Example 3: Run with verbose output to see detailed progress
# .\\Export-CheckedOutFiles.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/LegalHub" -Verbose

# Example 4: Run with WhatIf to preview changes (not applicable for this read-only script)
# Note: This script only reads data, so WhatIf is not applicable
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell

#Config variables
$tenantAdminURL = "https://contoso-admin.sharepoint.com"
$hubSiteURL = "https://contoso.sharepoint.com/sites/LegalHub"
$reportOutput = "C:\Temp\CheckedOutFiles.csv"

#Connect to SharePoint Online admin site 
$cred = Get-Credential
Connect-PnPOnline -Url $tenantAdminURL -Credentials $cred

#Get SPO sites
$spoSites = Get-PnPTenantSite -Detailed 

#Get hub site
$hubSite = Get-PnPHubSite -Identity $hubSiteURL
 
#Get associated sites with hub
$associatedSites = $spoSites | Where-Object { $_.HubSiteId -eq $hubSite.Id }

#Iterate through associated sites to find Checked-out files
if ($associatedSites -and $associatedSites.Count -gt 0) {
    ForEach ($site in $associatedSites) {
        if ($hubSite.SiteUrl -ne $site.Url) {
            Try {
                #Connect to the site
                Write-Host "Connect to : " $site.Url -f Green
                $siteConn = Connect-PnPOnline -Url $site.Url -Credentials $cred -ReturnConnection
 
                #Get all document libraries from the site
                $documentLibraries = Get-PnPList -Connection $siteConn | Where-Object { $_.BaseType -eq "DocumentLibrary" -and $_.ItemCount -gt 0 -and $_.Hidden -eq $False }
 
                #Iterate through document libraries in site
                ForEach ($library in $documentLibraries) {
                    
                    Write-host "Checking Library : " $library.Title -f Yellow

                    #CAML Query to filter Checked-out files
                    $query = "<View Scope='RecursiveAll'><Query><Where><IsNotNull><FieldRef Name='CheckoutUser' /></IsNotNull></Where></Query></View>"

                    #Get all Checked-out files of the library
                    $checkedOutFiles = Get-PnPListItem -List $library -Query $query
     
                    #Get details of each checked-out file
                    $results = @()                    
                    ForEach ($file in $checkedOutFiles) {
                        $results += [PSCustomObject][ordered]@{
                            LibraryName     = $library.Title
                            CheckedOutTo    = $file.FieldValues.CheckoutUser.LookupValue
                            CheckedOutSince = ($file.FieldValues.Last_x0020_Modified -as [datetime]).DateTime                           
                            FileSizeMB      = [Math]::Round((($file.FieldValues.File_x0020_Size/1024)/1024),2)
                            FileName        = $file.FieldValues.FileLeafRef
                            FileURL         = $file.FieldValues.FileDirRef
                        }
                        
                        #Export Checked out files data to CSV
                        $results | Export-Csv -Path $reportOutput -Append -NoTypeInformation
                    }                                            
                }
                                           
                Disconnect-PnPOnline -Connection $siteConn
            }
            Catch {
                write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
            }
        }        
    }   
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]


## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Arash Aghajani](https://github.com/arashaghajani) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/spo-export-checked-out-files-in-all-sites-associated-with-a-hub-site-to-csv" aria-hidden="true" />

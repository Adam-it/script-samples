

# Copy a library view across multiple libraries in destination site 

## Summary
I had a requirement to create a flat view "checked out files by me" across all libraries to enable users to easily spot checked out files by them within folders. I created the view in one library using the SharePoint Online UI and I used the script to copy the view to all libraries in another site. The url of the view needed to be without spaces so I created the view with a name without spaces and amended the title of the view using the script.

The sample script using PnP PowerShell to copy a library view from the source site and create it in libraries present in destination site.

Please refactor according to requirements as in the sample given only fields, views, query, item limit and certain settings are copied across.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "Source site URL where the view exists")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$SourceSiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Source library title from which to copy the view")]
    [string]$SourceListTitle,

    [Parameter(Mandatory = $true, HelpMessage = "Name of the view to copy")]
    [string]$SourceViewName,

    [Parameter(Mandatory = $true, HelpMessage = "Destination site URL where views will be created")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$DestinationSiteUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Output path for CSV report and transcript")]
    [string]$OutputPath = (Get-Location).Path,

    [Parameter(Mandatory = $false, HelpMessage = "View scope: Default (0), Recursive (1), RecursiveAll (2), FilesOnly (3)")]
    [ValidateSet(0, 1, 2, 3)]
    [int]$ViewScope = 1
)

begin {
    $dateTime = (Get-Date).ToString("yyyy-MM-dd_HHmmss")
    $transcriptPath = Join-Path $OutputPath "CopyLibraryView_Transcript_$dateTime.log"
    $csvPath = Join-Path $OutputPath "CopyLibraryView_Report_$dateTime.csv"

    Start-Transcript -Path $transcriptPath
    Write-Host "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] Starting library view copy operation" -ForegroundColor Cyan

    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath)) {
            Stop-Transcript
            throw "Output path '$OutputPath' does not exist. Please provide a valid path."
        }
    }

    Write-Verbose "Ensuring Microsoft 365 login..."
    m365 login --ensure

    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to authenticate with Microsoft 365. Exit code: $LASTEXITCODE"
    }

    Write-Host "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] Successfully authenticated" -ForegroundColor Green

    $script:ReportCollection = @()
    $script:Summary = @{
        TotalLibraries = 0
        ViewsCreated   = 0
        Skipped        = 0
        Failures       = 0
    }

    $SystemLists = @(
        "Converted Forms", "Master Page Gallery", "Customized Reports", 
        "Form Templates", "List Template Gallery", "Theme Gallery",
        "Reporting Templates", "Solution Gallery", "Style Library", 
        "Web Part Gallery", "Site Assets", "wfpub", "Site Pages", 
        "Images", "MicroFeed", "Pages"
    )
}

process {
    try {
        Write-Host "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] Retrieving source view from '$SourceListTitle' at $SourceSiteUrl" -ForegroundColor Cyan
        Write-Verbose "Executing: m365 spo list view get --webUrl $SourceSiteUrl --listTitle $SourceListTitle --viewTitle $SourceViewName --output json"

        $sourceViewJson = m365 spo list view get --webUrl $SourceSiteUrl --listTitle $SourceListTitle --viewTitle $SourceViewName --output json

        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve source view '$SourceViewName' from list '$SourceListTitle'. Exit code: $LASTEXITCODE"
        }

        $sourceView = $sourceViewJson | ConvertFrom-Json
        [xml]$listViewXML = $sourceView.ListViewXml

        $fieldsArr = @()
        $listViewXML.View.ViewFields.FieldRef | ForEach-Object {
            $fieldsArr += $_.Name
        }

        $sourceInternalName = $SourceViewName -replace '\s', ''

        Write-Host "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] Source view retrieved successfully. Fields: $($fieldsArr.Count), RowLimit: $($sourceView.RowLimit)" -ForegroundColor Green
        Write-Verbose "View fields: $($fieldsArr -join ', ')"

        Write-Host "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] Retrieving document libraries from $DestinationSiteUrl" -ForegroundColor Cyan
        Write-Verbose "Executing: m365 spo list list --webUrl $DestinationSiteUrl --output json"

        $listsJson = m365 spo list list --webUrl $DestinationSiteUrl --output json

        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve lists from destination site. Exit code: $LASTEXITCODE"
        }

        $allLists = $listsJson | ConvertFrom-Json
        $documentLibraries = $allLists | Where-Object {
            $_.BaseTemplate -eq 101 -and 
            $_.Hidden -eq $false -and 
            $SystemLists -notcontains $_.Title
        }

        $script:Summary.TotalLibraries = $documentLibraries.Count
        Write-Host "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] Found $($script:Summary.TotalLibraries) document libraries to process" -ForegroundColor Cyan

        foreach ($list in $documentLibraries) {
            try {
                Write-Verbose "Processing library: $($list.Title)"

                $existingViewJson = m365 spo list view get --webUrl $DestinationSiteUrl --listTitle $list.Title --viewTitle $SourceViewName --output json 2>$null

                if ($LASTEXITCODE -eq 0 -and $existingViewJson) {
                    Write-Verbose "View '$SourceViewName' already exists in library '$($list.Title)'. Skipping."
                    $script:Summary.Skipped++

                    $script:ReportCollection += [PSCustomObject]@{
                        Timestamp       = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                        SiteUrl         = $DestinationSiteUrl
                        LibraryName     = $list.Title
                        ViewName        = $SourceViewName
                        Status          = "Skipped - Already Exists"
                        ErrorMessage    = ""
                    }
                    continue
                }

                if ($PSCmdlet.ShouldProcess($list.Title, "Create view '$SourceViewName'")) {
                    Write-Verbose "Creating view '$sourceInternalName' in library '$($list.Title)'"

                    m365 spo list view add --webUrl $DestinationSiteUrl --listTitle $list.Title --title $sourceInternalName --fields ($fieldsArr -join ",") --rowLimit $sourceView.RowLimit --output json 2>&1 | Out-Null

                    if ($LASTEXITCODE -ne 0) {
                        throw "Failed to create view. Exit code: $LASTEXITCODE"
                    }

                    Write-Verbose "View created. Updating view properties (Title, ViewQuery, Scope)..."

                    $escapedViewQuery = $sourceView.ViewQuery -replace '"', '`"'

                    m365 spo list view set --webUrl $DestinationSiteUrl --listTitle $list.Title --viewTitle $sourceInternalName --Title $SourceViewName --ViewQuery $escapedViewQuery --Scope $ViewScope 2>&1 | Out-Null

                    if ($LASTEXITCODE -ne 0) {
                        throw "Failed to update view properties. Exit code: $LASTEXITCODE"
                    }

                    $script:Summary.ViewsCreated++
                    Write-Host "  [$(Get-Date -Format 'HH:mm:ss')] ✓ Created view in '$($list.Title)'" -ForegroundColor Green

                    $script:ReportCollection += [PSCustomObject]@{
                        Timestamp       = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                        SiteUrl         = $DestinationSiteUrl
                        LibraryName     = $list.Title
                        ViewName        = $SourceViewName
                        Status          = "Success"
                        ErrorMessage    = ""
                    }
                }
            }
            catch {
                $script:Summary.Failures++
                Write-Warning "Failed to process library '$($list.Title)': $($_.Exception.Message)"

                $script:ReportCollection += [PSCustomObject]@{
                    Timestamp       = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                    SiteUrl         = $DestinationSiteUrl
                    LibraryName     = $list.Title
                    ViewName        = $SourceViewName
                    Status          = "Failed"
                    ErrorMessage    = $_.Exception.Message
                }
                continue
            }
        }
    }
    catch {
        Write-Error "Critical error during view copy operation: $($_.Exception.Message)"
        Stop-Transcript
        throw
    }
}

end {
    if ($script:ReportCollection.Count -gt 0) {
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Force
        Write-Host "\n[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] CSV report exported to: $csvPath" -ForegroundColor Cyan
    }

    Write-Host "\n========================================" -ForegroundColor Cyan
    Write-Host "    SUMMARY - Copy Library View" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Total Libraries Processed : $($script:Summary.TotalLibraries)" -ForegroundColor White
    Write-Host "Views Created             : " -NoNewline
    Write-Host $script:Summary.ViewsCreated -ForegroundColor Green
    Write-Host "Skipped (Already Exists)  : " -NoNewline
    Write-Host $script:Summary.Skipped -ForegroundColor Yellow
    Write-Host "Failures                  : " -NoNewline
    if ($script:Summary.Failures -gt 0) {
        Write-Host $script:Summary.Failures -ForegroundColor Red
    } else {
        Write-Host $script:Summary.Failures -ForegroundColor Green
    }
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Transcript: $transcriptPath" -ForegroundColor Gray
    Write-Host "========================================\n" -ForegroundColor Cyan

    Stop-Transcript
}

# Usage Examples:
#
# Example 1: Basic usage - Copy view from source to destination
# .\CopyLibraryView.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/source" -SourceListTitle "Documents" -SourceViewName "Checked Out Files" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/destination"
#
# Example 2: Test with WhatIf to preview changes
# .\CopyLibraryView.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/source" -SourceListTitle "Documents" -SourceViewName "Checked Out Files" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/destination" -WhatIf
#
# Example 3: With verbose output and custom view scope
# .\CopyLibraryView.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/source" -SourceListTitle "Documents" -SourceViewName "All Items" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/destination" -ViewScope 2 -Verbose
#
# Example 4: Custom output path for reports
# .\CopyLibraryView.ps1 -SourceSiteUrl "https://contoso.sharepoint.com/sites/source" -SourceListTitle "Documents" -SourceViewName "Checked Out Files" -DestinationSiteUrl "https://contoso.sharepoint.com/sites/destination" -OutputPath "C:\\Reports"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell
  $SourceSite = Read-Host "Enter the source site url from which to copy the view from" #e.g.https://contoso.sharepoint.com/sites/Team1
$SourceList = Read-Host "Enter the source Library from which to copy the view from" #Demo Library
$SourceViewName = Read-Host "Enter the view name to be copied from" #Checked Out Files

$destSiteUrl = Read-Host "Enter the destination site url to which to copy the view to" #e.g.https://contoso.sharepoint.com/sites/testDemo

$dateTime = (Get-Date).toString("dd-MM-yyyy")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "CheckedOutViewCreationReport-" + $dateTime + ".csv"
$OutPutView = $directorypath + $fileName

#Arry to Skip System Lists and Libraries
$SystemLists = @("Converted Forms", "Master Page Gallery", "Customized Reports", "Form Templates", "List Template Gallery", "Theme Gallery",
                            "Reporting Templates", "Solution Gallery", "Style Library", "Web Part Gallery","Site Assets", "wfpub", "Site Pages", "Images", "MicroFeed","Pages")

#remove any spaces from view name
$SourceInternalName = $SourceViewName -replace '\s',''

Connect-PnPOnline -Url $SourceSite -Interactive

$CheckedOutView = Get-PnPView -List $SourceList -Identity $SourceViewName -Includes RowLimit, ViewQuery, ViewFields

#Array to Hold Result - PSObjects

$ViewCollection = @()

$fieldsArr = @();

$CheckedOutView.ViewFields |  ForEach-Object {
$fieldsArr +=$_;
}

#Flat view parameter
$viewScope=[Microsoft.SharePoint.Client.ViewScope]::Recursive 

Connect-PnPOnline -Url $destSiteUrl -UseWebLogin

#retrieving only document libraries from destination libraries
  foreach ($list in (Get-PnPList | ? {$_.BaseTemplate -eq 101 -and $_.Hidden -eq $false -and $SystemLists -notcontains $_.Title})) {
   $viewInL =  Get-PnPView -List $list.Title -Identity $SourceViewName -ErrorAction SilentlyContinue
   #create view only if not present
   if(!$viewInl)
   {

     $ExportVw = New-Object PSObject
     $ExportVw | Add-Member -MemberType NoteProperty -name "Site URL" -value $destSiteUrl
     $ExportVw | Add-Member -MemberType NoteProperty -name "Library Name" -value $list.Title
     $ExportVw | Add-Member -MemberType NoteProperty -name "View Name" -value $SourceViewName

    #create view with name without spaces and settings from source view 
     Add-PnPView -List $list.Title -Title $SourceInternalName -Query $CheckedOutView.ViewQuery -Fields $fieldsArr -RowLimit $CheckedOutView.RowLimit
     $viewInL =  Get-PnPView -List $list.Title -Identity $SourceInternalName -ErrorAction SilentlyContinue

     if($viewInL)
     {
       # Update the list view name to the display name and change the scope to recursive so that all files are displayed without any folders.
      Set-PnPView -List $list.Title -Identity $SourceInternalName -Values @{Scope=$viewScope;Title=$SourceViewName}   
     }
      $ViewCollection += $ExportVw
    }
   }

#Export the result Array to CSV file
$ViewCollection | Export-CSV $OutPutView -Force -NoTypeInformation

Disconnect-PnPOnline
 
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011)|
| [Adam Wójcik](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-copy-library-view" aria-hidden="true" />

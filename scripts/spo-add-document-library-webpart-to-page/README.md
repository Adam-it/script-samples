

# Add a document library web part to a page (and only show a specific folder)

## Summary

A customer had the requirement to create a page for each of their 86 folders in a document library so they could add more information on those topics. That meant creating 86 pages, each with a document library web part on it that showed a specific folder.

![Example Screenshot](assets/example.png)

The sample creating the page, adding the web parts and includes repeating this for all 86 folders. There is probably a really nice way to, in code, get all folders from the document library and loop through them. So I exported the document library to Excel and copied the folder names. I added some quotes and a comma (in an Excel formula using =CHAR(34) &  A2 & CHAR(34) &”,”) and added an array to store these.

# [PnP PowerShell](#tab/pnpps)
```powershell
Connect-PnPOnline -Url https://yourtenant.sharepoint.com/sites/Yoursite/ -Interactive
$ray = "folder1",
       "folder2",
       "folder3"

foreach ($name in $ray) {

    #create page
    Add-PnPPage -Name $name -LayoutType Article -HeaderLayoutType NoImage -CommentsEnabled:$false
    
    #add sections
    Add-PnPPageSection -Page $name -SectionTemplate TwoColumn -Order 1
    
    #add text webpart
    Add-PnPPageTextPart -Page $name -Section 1 -Column 1 -Text "This is $name"
    
    #add doclib
    $DocLib = Get-PnPList -Identity Documents
    $DocLibID = $DocLib.id.tostring()
    Add-PnPPageWebPart -Page $name -DefaultWebPartType List -Section 1 -Column 1 -WebPartProperties @{isDocumentLibrary="true";selectedListId="$($DocLibID)";selectedFolderPath="/$name";hideCommandBar="false"}
    $page = Get-PnPPage -Identity $name
    $page.Publish()
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
  [Parameter(Mandatory = $true, HelpMessage = "URL of the SharePoint site where pages will be created")]
  [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)(/.*)?$')]
  [string]$SiteUrl,

  [Parameter(Mandatory = $true, HelpMessage = "Name of the document library to link in the web part")]
  [string]$DocumentLibraryName = "Documents",

  [Parameter(Mandatory = $true, HelpMessage = "Array of folder names to create pages for")]
  [ValidateNotNullOrEmpty()]
  [string[]]$FolderNames,

  [Parameter(Mandatory = $false, HelpMessage = "Path to save transcript log")]
  [string]$OutputPath = (Get-Location).Path
)

begin {
  # Start transcript
  $timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
  $transcriptPath = Join-Path $OutputPath "spo-add-doclib-webpart-$timestamp.log"
  Start-Transcript -Path $transcriptPath | Out-Null
  Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Starting script execution..." -ForegroundColor Cyan

  # Initialize summary counters
  $script:Summary = @{
    Total   = $FolderNames.Count
    Created = 0
    Skipped = 0
    Failed  = 0
  }

  # Ensure user is logged in
  Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Ensuring CLI for Microsoft 365 session..." -ForegroundColor Cyan
  m365 login --ensure
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to authenticate with CLI for Microsoft 365. Please check your credentials."
  }
  Write-Host "[$(Get-Date -Format 'HH:mm:ss')] ✓ Authentication successful" -ForegroundColor Green

  # Get document library ID dynamically
  Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Retrieving document library '$DocumentLibraryName'..." -ForegroundColor Cyan
  try {
    $libraryJson = m365 spo list get --title $DocumentLibraryName --webUrl $SiteUrl --output json
    if ($LASTEXITCODE -ne 0) {
      throw "Failed to retrieve document library '$DocumentLibraryName'. Please verify the library exists."
    }
    $library = $libraryJson | ConvertFrom-Json
    $script:LibraryId = $library.Id
    Write-Host "[$(Get-Date -Format 'HH:mm:ss')] ✓ Document library ID: $script:LibraryId" -ForegroundColor Green
  }
  catch {
    Write-Error "Error retrieving document library: $_"
    throw
  }
}

process {
  foreach ($folderName in $FolderNames) {
    $fileName = "$folderName.aspx"
    Write-Verbose "Processing folder: $folderName (Page: $fileName)"

    if ($PSCmdlet.ShouldProcess($fileName, "Create page with document library web part for folder '$folderName'")) {
      try {
        # Create page
        Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Creating page '$fileName'..." -ForegroundColor Cyan
        m365 spo page add --name $fileName --title $folderName --webUrl $SiteUrl --output json | Out-Null
        if ($LASTEXITCODE -ne 0) {
          throw "Failed to create page '$fileName'"
        }

        # Add two-column section
        Write-Verbose "Adding two-column section to page '$fileName'"
        m365 spo page section add --pageName $fileName --webUrl $SiteUrl --sectionTemplate TwoColumn --order 1 --output json | Out-Null
        if ($LASTEXITCODE -ne 0) {
          throw "Failed to add section to page '$fileName'"
        }

        # Add text web part
        Write-Verbose "Adding text web part to page '$fileName'"
        m365 spo page text add --webUrl $SiteUrl --pageName $fileName --text "This is $folderName" --section 1 --column 1 --output json | Out-Null
        if ($LASTEXITCODE -ne 0) {
          throw "Failed to add text web part to page '$fileName'"
        }

        # Prepare web part properties (dynamic library ID + folder path)
        $webPartPropsObject = @{
          isDocumentLibrary  = "true"
          selectedListId     = $script:LibraryId
          selectedFolderPath = "/$folderName"
          hideCommandBar     = "false"
        }
        $webPartPropsJson = ($webPartPropsObject | ConvertTo-Json -Compress).Replace('"', '\"')

        # Add document library web part
        Write-Verbose "Adding document library web part to page '$fileName'"
        m365 spo page clientsidewebpart add --webUrl $SiteUrl --pageName $fileName --standardWebPart List --section 1 --column 1 --webPartProperties $webPartPropsJson --output json | Out-Null
        if ($LASTEXITCODE -ne 0) {
          throw "Failed to add document library web part to page '$fileName'"
        }

        # Publish page
        Write-Verbose "Publishing page '$fileName'"
        m365 spo page set --name $fileName --webUrl $SiteUrl --publish --output json | Out-Null
        if ($LASTEXITCODE -ne 0) {
          throw "Failed to publish page '$fileName'"
        }

        Write-Host "[$(Get-Date -Format 'HH:mm:ss')] ✓ Successfully created and published page '$fileName'" -ForegroundColor Green
        $script:Summary.Created++
      }
      catch {
        Write-Warning "Failed to create page for folder '$folderName': $_"
        $script:Summary.Failed++
        continue
      }
    }
    else {
      Write-Host "[$(Get-Date -Format 'HH:mm:ss')] [WhatIf] Would create page '$fileName' for folder '$folderName'" -ForegroundColor Yellow
      $script:Summary.Skipped++
    }
  }
}

end {
  # Display summary
  Write-Host "`n========================================" -ForegroundColor Cyan
  Write-Host "           EXECUTION SUMMARY            " -ForegroundColor Cyan
  Write-Host "========================================" -ForegroundColor Cyan
  Write-Host "Total folders processed: $($script:Summary.Total)" -ForegroundColor Gray
  Write-Host "Pages created:           " -NoNewline -ForegroundColor Gray
  if ($script:Summary.Created -gt 0) {
    Write-Host $script:Summary.Created -ForegroundColor Green
  } else {
    Write-Host $script:Summary.Created -ForegroundColor Gray
  }
  Write-Host "Skipped (WhatIf):        $($script:Summary.Skipped)" -ForegroundColor Yellow
  Write-Host "Failed:                  " -NoNewline -ForegroundColor Gray
  if ($script:Summary.Failed -gt 0) {
    Write-Host $script:Summary.Failed -ForegroundColor Red
  } else {
    Write-Host $script:Summary.Failed -ForegroundColor Gray
  }
  Write-Host "========================================`n" -ForegroundColor Cyan

  Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Script execution completed" -ForegroundColor Cyan
  Write-Host "Transcript saved: $transcriptPath" -ForegroundColor Gray
  Stop-Transcript | Out-Null
}

# Example 1: Create pages for three folders (basic usage)
# .\Create-DocLibPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -DocumentLibraryName "Documents" -FolderNames @("Branding", "Campaigns", "Reports")

# Example 2: Test with WhatIf to preview changes without making them
# .\Create-DocLibPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -DocumentLibraryName "Documents" -FolderNames @("Branding", "Campaigns", "Reports") -WhatIf

# Example 3: Create pages with verbose output for detailed logging
# .\Create-DocLibPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -DocumentLibraryName "Documents" -FolderNames @("Branding", "Campaigns", "Reports") -Verbose

# Example 4: Create pages for custom library and save transcript to specific path
# .\Create-DocLibPages.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -DocumentLibraryName "Policies" -FolderNames @("Benefits", "Onboarding", "Training") -OutputPath "C:\Logs"
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Source Credit

Sample first appeared on [Use PnP Powershell to add a document library webpart to a page (and only show a specific folder) | Tech Community](https://techcommunity.microsoft.com/t5/microsoft-365-pnp-blog/use-pnp-powershell-to-add-a-document-library-webpart-to-a-page/ba-p/2428310)

## Contributors

| Author(s) |
|-----------|
| Marijn Somers |
| [Adam Wójcik](https://github.com/Adam-it)|
| [Todd Klindt](https://www.toddklindt.com)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-add-document-library-webpart-to-page" aria-hidden="true" />

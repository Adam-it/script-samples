

# Update content type of files in folder with system update

## Summary

Update content type with system update option for all files in a folder within a library to a custom content type to avoid updating modified and modified by properties.

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Copy a script  below


# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL (e.g., https://contoso.sharepoint.com/sites/site)")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "List title (e.g., 'Documents')")]
    [string]$ListTitle,

    [Parameter(Mandatory = $true, HelpMessage = "Folder server-relative path (e.g., '/sites/Site/Shared Documents/Folder')")]
    [string]$FolderPath,

    [Parameter(Mandatory = $true, HelpMessage = "Content type name to apply (e.g., 'Legal')")]
    [string]$ContentTypeName,

    [Parameter(Mandatory = $false, HelpMessage = "Output path for transcript and optional CSV report")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    # Ensure user is signed in
    Write-Verbose "Ensuring CLI for Microsoft 365 session is active..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to CLI for Microsoft 365. Please check your authentication."
    }

    # Initialize summary
    $script:Summary = @{
        TotalFiles = 0
        Updated    = 0
        Failures   = 0
    }

    # Start transcript
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "UpdateContentType_$timestamp.log"
    Start-Transcript -Path $transcriptPath | Out-Null
    Write-Verbose "Transcript started: $transcriptPath"
}

process {
    try {
        Write-Host "Retrieving files from folder: $FolderPath" -ForegroundColor Cyan

        # Retrieve files from folder using OData filter
        $filesJson = m365 spo listitem list --webUrl $SiteUrl --listTitle $ListTitle --filter "startswith(FileRef,'$FolderPath') and FileSystemObjectType eq 0" --fields "Id,FileRef" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve files from folder '$FolderPath'. CLI: $filesJson"
        }

        $files = @($filesJson | ConvertFrom-Json)
        $script:Summary.TotalFiles = $files.Count

        if ($files.Count -eq 0) {
            Write-Warning "No files found in folder: $FolderPath"
            return
        }

        Write-Host "Found $($files.Count) file(s). Updating content type to '$ContentTypeName'..." -ForegroundColor Cyan

        # Update each file's content type
        foreach ($file in $files) {
            try {
                Write-Verbose "Updating file: $($file.FileRef)"

                $updateResult = m365 spo listitem set --webUrl $SiteUrl --listTitle $ListTitle --id $file.Id --contentType $ContentTypeName --systemUpdate --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to update file '$($file.FileRef)'. CLI: $updateResult"
                    $script:Summary.Failures++
                    continue
                }

                $script:Summary.Updated++
            }
            catch {
                Write-Warning "Error updating file '$($file.FileRef)': $_"
                $script:Summary.Failures++
                continue
            }
        }
    }
    catch {
        Write-Error "Critical error: $_"
        throw
    }
}

end {
    Stop-Transcript | Out-Null

    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "CONTENT TYPE UPDATE SUMMARY" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Total files found: $($script:Summary.TotalFiles)" -ForegroundColor White
    Write-Host "Successfully updated: $($script:Summary.Updated)" -ForegroundColor Green
    Write-Host "Failed: $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "Content type applied: $ContentTypeName" -ForegroundColor White
    Write-Host "Folder path: $FolderPath" -ForegroundColor White
    Write-Host "Transcript saved: $transcriptPath" -ForegroundColor White
    Write-Host "========================================`n" -ForegroundColor Cyan
}

# Usage examples:
#
# Example 1: Basic usage with verbose output
# ./UpdateContentTypeSystemUpdate.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Documents" -FolderPath "/sites/project/Shared Documents/Legal" -ContentTypeName "Legal" -Verbose
#
# Example 2: Custom output path
# ./UpdateContentTypeSystemUpdate.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Documents" -FolderPath "/sites/project/Shared Documents/HR" -ContentTypeName "HR Document" -OutputPath "C:\Reports"
#
# Example 3: Update content type for nested folder
# ./UpdateContentTypeSystemUpdate.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Documents" -FolderPath "/sites/project/Shared Documents/Archives/2023" -ContentTypeName "Archived Document"
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)
```powershell

#Config Variables
$SiteURL = "https://tenant.sharepoint.com/sites/Estimator"
$ListName = "Documents" 
$FolderServerRelativePath= "/sites/Estimator/Shared Documents/LineManagement"
$NewContentType = "Legal"

Connect-PnPOnline -url $SiteURL  -Interactive
 
Try {

  #Get all files from folder
   Get-PnPListItem -List $ListName -PageSize 2000 | Where {$_.FieldValues.FileRef -like "$FolderServerRelativePath*" -and $_.FileSystemObjectType -eq "File"  } | ForEach-Object {
    Write-host $_.FieldValues.FileRef
   Set-PnPListItem -UpdateType SystemUpdate -List  $ListName -ContentType $NewContentType -Identity $_
  }
}
catch {
    write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
}
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshme011) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-list-update-contenttype-systemupdate" aria-hidden="true" />

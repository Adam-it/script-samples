

# Export all site pages details from Site Pages library

## Summary

This sample will export the required site pages information to CSV. Available for both PnP PowerShell and CLI for Microsoft 365.

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as below,
- First, we will connect to the site from which site we want to get Site Pages library details.
    - Then we will get Site Pages details and export it to CSV.

# [PnP PowerShell](#tab/pnpps)
```powershell
$siteURL = "https://domain.sharepoint.com/"
$username = "username@domain.onmicrosoft.com"
$password = "********"
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd
$dateTime = "_{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$basePath = "E:\Contribution\PnP-Scripts\Logs\"
$csvPath = $basePath + "\SitePages" + $dateTime + ".csv"
$global:sitePagesCollection = @()

Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $creds)     
    Write-Host "Connecting to Site '$($siteURL)'" -f Yellow   
    Connect-PnPOnline -Url $siteURL -Credential $creds
    Write-Host "Connection Successful" -f Green 
}

Function GetSitePagesDetails {    
    try {
        Write-Host "Getting site pages information..."  -ForegroundColor Yellow 
        $sitePages = Get-PnPListItem -List "Site Pages"     
        ForEach ($Page in $sitePages) {
            $sitePagesInfo = New-Object PSObject -Property ([Ordered] @{
                    'ID'               = $Page.ID
                    'Title'            = $Page.FieldValues.Title
                    'Description'      = $Page.FieldValues.Description
                    'Page Layout Type' = $Page.FieldValues.PageLayoutType
                    'FileRef'          = $Page.FieldValues.FileRef  
                    'FileLeafRef'      = $Page.FieldValues.FileLeafRef      
                    'Created'          = Get-Date -Date $Page.FieldValues.Created_x0020_Date -Format "dddd MM/dd/yyyy HH:mm"
                    'Modified'         = Get-Date -Date $Page.FieldValues.Last_x0020_Modified -Format "dddd MM/dd/yyyy HH:mm"
                    'Modified By'      = $Page.FieldValues.Modified_x0020_By
                    'Created By'       = $Page.FieldValues.Created_x0020_By
                    'Author'           = $Page.FieldValues.Author.Email
                    'Editor'           = $Page.FieldValues.Editor.Email
                    'BannerImage Url'  = $Page.FieldValues.BannerImageUrl.Url   
                    'File_x0020_Type'  = $Page.FieldValues.File_x0020_Type   
                })
            $global:sitePagesCollection += $sitePagesInfo
        }
    }
    catch {
        Write-Host "Error in getting site pages:" $_.Exception.Message -ForegroundColor Red                 
    }
    Write-Host "Exporting to CSV..."  -ForegroundColor Yellow 
    $global:SitePagesCollection | Export-Csv $csvPath -NoTypeInformation -Append
    Write-Host "Exported to CSV successfully!"  -ForegroundColor Green	

}

Function StartProcessing {
    Login($creds);
    GetSitePagesDetails
}

StartProcessing
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Export-SPOSitePagesDetails {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory = $true, HelpMessage = "The URL of the SharePoint site")]
        [string]$SiteUrl,

        [Parameter(HelpMessage = "The base directory path where the CSV file will be saved")]
        [string]$OutputPath = (Get-Location).Path,

        [Parameter(HelpMessage = "Export results to a CSV file")]
        [switch]$ExportToCsv
    )

    begin {
        # Login to Microsoft 365
        Write-Verbose "Ensuring Microsoft 365 login..."
        m365 login --ensure 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to authenticate to Microsoft 365. Please run 'm365 login' first."
        }
        Write-Verbose "Successfully authenticated to Microsoft 365"

        # Initialize summary and collection
        $script:Summary = @{
            PagesFound    = 0
            PagesExported = 0
            Failures      = 0
        }
        $script:PagesCollection = @()
    }

    process {
        try {
            # Get all site pages from the Site Pages library
            Write-Verbose "Retrieving site pages from '$SiteUrl'..."
            $pagesJson = m365 spo page list --webUrl $SiteUrl --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve site pages. CLI output: $pagesJson"
            }

            $pages = @($pagesJson | ConvertFrom-Json)
            $script:Summary.PagesFound = $pages.Count
            Write-Verbose "Found $($pages.Count) site pages"

            Write-Host "Processing $($pages.Count) site pages..." -ForegroundColor Cyan

            # Process each page
            foreach ($page in $pages) {
                try {
                    # Map properties to match PnP PowerShell output format
                    $pageInfo = [PSCustomObject]@{
                        'ID'               = if ($page.ListItemAllFields.Id) { $page.ListItemAllFields.Id } else { $page.Id }
                        'Title'            = $page.Title
                        'Description'      = $page.Description
                        'Page Layout Type' = $page.PageLayoutType
                        'FileRef'          = $page.AbsoluteUrl
                        'FileLeafRef'      = $page.Name
                        'Created'          = if ($page.TimeCreated) { Get-Date -Date $page.TimeCreated -Format "dddd MM/dd/yyyy HH:mm" } else { "" }
                        'Modified'         = if ($page.TimeLastModified) { Get-Date -Date $page.TimeLastModified -Format "dddd MM/dd/yyyy HH:mm" } else { "" }
                        'Modified By'      = if ($page.ListItemAllFields.Editor) { $page.ListItemAllFields.Editor.Title } else { "" }
                        'Created By'       = if ($page.ListItemAllFields.Author) { $page.ListItemAllFields.Author.Title } else { "" }
                        'Author'           = if ($page.ListItemAllFields.Author.Email) { $page.ListItemAllFields.Author.Email } else { "" }
                        'Editor'           = if ($page.ListItemAllFields.Editor.Email) { $page.ListItemAllFields.Editor.Email } else { "" }
                        'BannerImage Url'  = if ($page.BannerImageUrl.Url) { $page.BannerImageUrl.Url } elseif ($page.BannerImageUrl) { $page.BannerImageUrl } else { "" }
                        'File_x0020_Type'  = if ($page.Name) { [System.IO.Path]::GetExtension($page.Name).TrimStart('.') } else { "" }
                    }
                    $script:PagesCollection += $pageInfo
                    $script:Summary.PagesExported++
                    Write-Verbose "Processed page: $($page.Title)"
                }
                catch {
                    Write-Warning "Failed to process page '$($page.Title)': $_"
                    $script:Summary.Failures++
                }
            }
        }
        catch {
            Write-Error "Failed to retrieve or process site pages: $_"
            $script:Summary.Failures++
        }
    }

    end {
        # Export to CSV if requested
        if ($ExportToCsv -and $script:PagesCollection.Count -gt 0) {
            $dateTime = "_{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
            $csvPath = Join-Path -Path $OutputPath -ChildPath ("SitePages" + $dateTime + ".csv")
            
            if ($PSCmdlet.ShouldProcess($csvPath, 'Export site pages to CSV')) {
                try {
                    $script:PagesCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
                    Write-Host "`nSite pages exported successfully to: $csvPath" -ForegroundColor Green
                }
                catch {
                    Write-Warning "Failed to export CSV to '$csvPath': $_"
                    $script:Summary.Failures++
                }
            }
        }
        elseif ($ExportToCsv -and $script:PagesCollection.Count -eq 0) {
            Write-Warning "No site pages found to export to CSV"
        }

        # Display summary
        Write-Host "`n========================================" -ForegroundColor Cyan
        Write-Host "         Site Pages Export Summary" -ForegroundColor Cyan
        Write-Host "========================================" -ForegroundColor Cyan
        Write-Host "Pages Found:    $($script:Summary.PagesFound)" -ForegroundColor White
        Write-Host "Pages Exported: $($script:Summary.PagesExported)" -ForegroundColor Green
        Write-Host "Failures:       $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
        Write-Host "========================================`n" -ForegroundColor Cyan

        # Display pages in terminal if not exporting to CSV
        if (-not $ExportToCsv -and $script:PagesCollection.Count -gt 0) {
            Write-Host "`nSite Pages Details:" -ForegroundColor Cyan
            $script:PagesCollection | Format-Table -AutoSize
        }
    }
}

# Example usage
Export-SPOSitePagesDetails -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -ExportToCsv -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***


## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati (https://github.com/chandaniprajapati) |
| Adam Wójcik (https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-all-site-pages-details" aria-hidden="true" />

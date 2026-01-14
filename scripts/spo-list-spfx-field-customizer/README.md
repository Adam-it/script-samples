# List all SPFx field customizer

## Summary

This is a simple PnP PowerShell script to list all SPFx field customizers in a SharePoint Online environment.

It's done by scarping your entire tenant, every site, and every field on every list, and checking if the `ClientSideComponentId` is set to a non-empty value, which indicates that the field has a customizer applied.


![Example Screenshot](assets/preview.png)

In light of the recent [announcement from Microsoft](https://support.microsoft.com/en-gb/office/support-update-for-sharepoint-framework-field-customizers-in-lists-and-document-libraries-0eccc64e-4512-47df-9da0-d855be22fb0a#) you might want to get a holistic picture of your tenants usage of Field Customizers.


# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint tenant admin URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$AdminUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Directory path for CSV export")]
    [ValidateScript({ Test-Path $_ -PathType Container })]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Verbose "Ensuring Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365"
    }
    Write-Verbose "Successfully authenticated to Microsoft 365"

    $script:ReportCollection = @()
    $script:Summary = @{
        TotalSites = 0
        TotalLists = 0
        TotalFieldsScanned = 0
        CustomizersFound = 0
        FailedSites = 0
    }

    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "SPFxFieldCustomizers_Transcript_$timestamp.log"
    Start-Transcript -Path $transcriptPath | Out-Null
    Write-Host "Transcript logging started: $transcriptPath" -ForegroundColor Cyan
}

process {
    Write-Host "Retrieving all sites from tenant..." -ForegroundColor Cyan
    
    try {
        $sitesJson = m365 spo site list --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sites: $sitesJson"
        }
        $sites = $sitesJson | ConvertFrom-Json
        $script:Summary.TotalSites = $sites.Count
        Write-Host "Found $($sites.Count) sites to scan" -ForegroundColor Green
    }
    catch {
        Write-Error "Failed to retrieve sites from tenant: $_"
        throw
    }

    foreach ($site in $sites) {
        Write-Verbose "Processing site: $($site.Url)"
        
        try {
            $listsJson = m365 spo list list --webUrl $site.Url --filter "Hidden eq false" --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve lists from $($site.Url): $listsJson"
                $script:Summary.FailedSites++
                continue
            }
            $lists = $listsJson | ConvertFrom-Json
            $script:Summary.TotalLists += $lists.Count
            Write-Verbose "Found $($lists.Count) lists in site $($site.Url)"

            foreach ($list in $lists) {
                Write-Verbose "Processing list: $($list.Title)"
                
                try {
                    $fieldsJson = m365 spo field list --webUrl $site.Url --listTitle $list.Title --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to retrieve fields from list '$($list.Title)' in site $($site.Url): $fieldsJson"
                        continue
                    }
                    $fields = $fieldsJson | ConvertFrom-Json
                    $script:Summary.TotalFieldsScanned += $fields.Count

                    $customizedFields = $fields | Where-Object { 
                        $_.ClientSideComponentId -ne "00000000-0000-0000-0000-000000000000" -and 
                        $null -ne $_.ClientSideComponentId 
                    }

                    foreach ($field in $customizedFields) {
                        $script:ReportCollection += [PSCustomObject]@{
                            SiteUrl = $site.Url
                            ListTitle = $list.Title
                            FieldTitle = $field.Title
                            FieldInternalName = $field.InternalName
                            ClientSideComponentId = $field.ClientSideComponentId
                            ClientSideComponentProperties = $field.ClientSideComponentProperties
                        }
                        $script:Summary.CustomizersFound++
                        Write-Verbose "Found SPFx field customizer: $($field.Title)"
                    }
                }
                catch {
                    Write-Warning "Error processing list '$($list.Title)' in site $($site.Url): $_"
                    continue
                }
            }
        }
        catch {
            Write-Warning "Error processing site $($site.Url): $_"
            $script:Summary.FailedSites++
            continue
        }
    }
}

end {
    if ($script:ReportCollection.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "SPFxFieldCustomizers_$timestamp.csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation
        Write-Host "CSV report exported to: $csvPath" -ForegroundColor Green
    }
    else {
        Write-Host "No SPFx field customizers found in the tenant." -ForegroundColor Yellow
    }

    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Total sites scanned: $($script:Summary.TotalSites)" -ForegroundColor White
    Write-Host "Total lists scanned: $($script:Summary.TotalLists)" -ForegroundColor White
   Write-Host "Total fields scanned: $($script:Summary.TotalFieldsScanned)" -ForegroundColor White
    Write-Host "SPFx field customizers found: $($script:Summary.CustomizersFound)" -ForegroundColor $(if ($script:Summary.CustomizersFound -gt 0) { 'Green' } else { 'Yellow' })
    Write-Host "Failed sites: $($script:Summary.FailedSites)" -ForegroundColor $(if ($script:Summary.FailedSites -gt 0) { 'Red' } else { 'Green' })
   Write-Host "============================`n" -ForegroundColor Cyan

    Stop-Transcript | Out-Null
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Cyan
}

# Example 1: Scan all sites and export to current directory
# .\List-SPFxFieldCustomizers.ps1 -AdminUrl "https://contoso-admin.sharepoint.com"

# Example 2: Export to specific directory
# .\List-SPFxFieldCustomizers.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\Reports"

# Example 3: Run with verbose output
# .\List-SPFxFieldCustomizers.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***


# [PnP PowerShell](#tab/pnpps)

```powershell

$AdminUrl = "https://<Tenant>-admin.sharepoint.com/";



$FieldCustomizers = @()

Connect-PnPOnline $AdminUrl # <Your connection parameters go here>

$sites = Get-PnPTenantSite

foreach($site in $sites){
    Write-Host "Processing site: $($site.Url)" -ForegroundColor Cyan
    $siteCon = Connect-PnPOnline -Url $site.Url -ReturnConnection # <Your connection parameters go here>

    $lists = Get-PnPList -Connection $siteCon
    foreach($list in $lists){
        Write-Host "Processing list: $($list.Title)" -ForegroundColor Yellow
        $fields = Get-PnPField -List $list -Connection $siteCon 

        foreach($field in $fields){
            if($field.ClientSideComponentId -ne [Guid]::Empty){
                $FieldCustomizers += [PSCustomObject]@{
                    SiteUrl = $site.Url
                    ListTitle = $list.Title
                    FieldTitle = $field.Title
                    FieldInternalName = $field.InternalName
                    ClientSideComponentId = $field.ClientSideComponentId
                    ClientSideComponentProperties = $field.ClientSideComponentProperties
                }
            }
        }
    }
}

# Export the results to a CSV file
if ($FieldCustomizers.Count -eq 0) {
    Write-Host "No field customizers found." -ForegroundColor Red
    return
}

Write-host $FieldCustomizers | Format-Table -AutoSize

$FieldCustomizers | Export-Csv -Path "FieldCustomizers.csv" -NoTypeInformation

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***


## Contributors

| Author(s)                       |
| ------------------------------- |
| [Adam Wójcik](https://github.com/Adam-it) |
| [Dan Toft](https://Dan-toft.dk) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-list-spfx-field-customizer" aria-hidden="true" />

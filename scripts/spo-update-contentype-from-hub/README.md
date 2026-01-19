---
plugin: add-to-gallery
---

# Update Content type from Hub on sites

## Summary

Back in the days Microsoft would update the content types in the content type hub and then push it to all site collections. This is no longer the case. You need to run a script to update the content types in all site collections.
This is such a script. It will check all site collections and update the content type if it exists.


![Example Screenshot](assets/example.png)


# [PnP PowerShell](#tab/pnpps)

```powershell

#when a content type in the Content type gallery is updated and republished
# it is not automaticly updated on the site collections where it is used.

# This script will check every site collection and update the content type

$adminUrl = "https://contoso-admin.sharepoint.com/"
$pnpClientId = "the client id of the PnP Rocks app"
$contenttypeName = "Contoso Project Document"

if(-not $conn)
{
    $conn = Connect-PnPOnline -Url $adminUrl -Interactive -ClientId $pnpClientId -ReturnConnection -WarningAction Ignore
}

$allsites = Get-PnPTenantSite -Connection $conn -ErrorAction Stop
foreach($site in $allsites)
{
    Write-Host "Site: $($site.Url)"
    try 
    {
        $siteUrl = $site.Url
        Connect-PnPOnline -Url $siteUrl -Interactive -ClientId $pnpClientId  -WarningAction Ignore
        $contenttype = Get-PnPContentType -ErrorAction SilentlyContinue -Identity $contenttypeName 
        if($contenttype)
        {
                Write-Host "Updating content type: $($contenttype.Name) at site: $siteUrl " -ForegroundColor Green
                Add-PnPContentTypesFromContentTypeHub -ContentTypes $contenttype.Id  -ErrorAction Stop
        }
        else
        {
            Write-Host "No content type: $($contenttype.Name) at site: $siteUrl " 
        }   
    }
    catch 
    {
            <#Do this if a terminating exception happens#>
            throw $_
    }
}


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint admin center URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$AdminUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Content type name or ID to sync")]
    [string]$ContentType,

    [Parameter(Mandatory = $false, HelpMessage = "URL pattern to filter sites (e.g., 'marketing' to match sites containing 'marketing')")]
    [string]$SiteUrlFilter,

    [Parameter(Mandatory = $false, HelpMessage = "Output folder for CSV and transcript")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath)) {
            throw "Output path '$OutputPath' does not exist"
        }
    }

    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to login to Microsoft 365. Please run 'm365 login' first."
    }

    $script:Summary = @{
        SitesChecked = 0
        SitesUpdated = 0
        SitesSkipped = 0
        Failures     = 0
    }

    $script:ReportCollection = @()

    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    Start-Transcript -Path "$OutputPath\ContentTypeSync_$timestamp.log"

    Write-Host "Retrieving all tenant sites..." -ForegroundColor Cyan
    
    if ($SiteUrlFilter) {
        Write-Host "Filtering sites by URL pattern: *$SiteUrlFilter*" -ForegroundColor White
        $sitesJson = m365 spo site list --filter "Url -like '$SiteUrlFilter'" --output json 2>&1
    }
    else {
        $sitesJson = m365 spo site list --output json 2>&1
    }
    
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve sites: $sitesJson"
    }
    $script:AllSites = @($sitesJson | ConvertFrom-Json)
    Write-Host "Found $($script:AllSites.Count) sites" -ForegroundColor White
}

process {
    foreach ($site in $script:AllSites) {
        try {
            $script:Summary.SitesChecked++
            Write-Verbose "Checking site: $($site.Url)"

            $ctJson = m365 spo contenttype get --webUrl $site.Url --name $ContentType --output json 2>&1

            if ($LASTEXITCODE -ne 0) {
                Write-Verbose "Content type not found on $($site.Url)"
                $script:Summary.SitesSkipped++
                $script:ReportCollection += [PSCustomObject]@{
                    SiteUrl         = $site.Url
                    SiteTitle       = $site.Title
                    ContentTypeName = $ContentType
                    ContentTypeId   = "N/A"
                    Status          = "Skipped (Not Found)"
                    ErrorMessage    = ""
                    Timestamp       = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                }
                continue
            }

            $ct = $ctJson | ConvertFrom-Json
            Write-Host "  Found '$($ct.Name)' (ID: $($ct.Id.StringValue)) on $($site.Url)" -ForegroundColor White

            if ($PSCmdlet.ShouldProcess("$($site.Url) - $($ct.Name)", "Sync content type from hub")) {
                Write-Host "  Syncing content type..." -ForegroundColor Yellow
                m365 spo contenttype sync --webUrl $site.Url --id $ct.Id.StringValue 2>&1 | Out-Null

                if ($LASTEXITCODE -ne 0) {
                    throw "Sync failed"
                }

                Write-Host "  SUCCESS: Synced content type" -ForegroundColor Green
                $script:Summary.SitesUpdated++
                $script:ReportCollection += [PSCustomObject]@{
                    SiteUrl         = $site.Url
                    SiteTitle       = $site.Title
                    ContentTypeName = $ct.Name
                    ContentTypeId   = $ct.Id.StringValue
                    Status          = "Synced"
                    ErrorMessage    = ""
                    Timestamp       = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                }
            }
        }
        catch {
            Write-Warning "Failed to sync on $($site.Url): $($_.Exception.Message)"
            $script:Summary.Failures++
            $script:ReportCollection += [PSCustomObject]@{
                SiteUrl         = $site.Url
                SiteTitle       = $site.Title
                ContentTypeName = if ($ct) { $ct.Name } else { $ContentType }
                ContentTypeId   = if ($ct) { $ct.Id.StringValue } else { "N/A" }
                Status          = "Failed"
                ErrorMessage    = $_.Exception.Message
                Timestamp       = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
            }
        }
    }
}

end {
    Stop-Transcript

    $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
    $csvPath = "$OutputPath\ContentTypeSync_$timestamp.csv"
    $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation
    Write-Host "`nCSV report exported to: $csvPath" -ForegroundColor Cyan

    Write-Host "`n===== Summary =====" -ForegroundColor Cyan
    Write-Host "Sites Checked: $($script:Summary.SitesChecked)" -ForegroundColor White
    Write-Host "Sites Updated: $($script:Summary.SitesUpdated)" -ForegroundColor Green
    Write-Host "Sites Skipped: $($script:Summary.SitesSkipped)" -ForegroundColor White
    Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'White' })
}

# Sync content type by name with WhatIf
# .\Sync-ContentTypeFromHub.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -ContentType "Contoso Project Document" -WhatIf

# Sync content type by ID
# .\Sync-ContentTypeFromHub.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -ContentType "0x01007926A45D687BA842B947286090B8F67D"

# Sync only on sites with 'marketing' in URL
# .\Sync-ContentTypeFromHub.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -ContentType "Contoso Project Document" -SiteUrlFilter "marketing"

# Sync with custom output path
# .\Sync-ContentTypeFromHub.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -ContentType "Contoso Project Document" -OutputPath "C:\Logs"

# Sync with verbose output
# .\Sync-ContentTypeFromHub.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -ContentType "Contoso Project Document" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |
| Reshmee Auckloo |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-contentype-from-hub" aria-hidden="true" />

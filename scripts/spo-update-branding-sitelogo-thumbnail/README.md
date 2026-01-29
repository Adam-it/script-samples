

# Updates SharePoint Site Logo and Thumbnail

## Summary

In SharePoint Online sites, the distinction between the `Site Logo` and `Site Thumbnail` is crucial. The site logo appears in the site header, while the site thumbnail is used in search results, site cards, file copying/moving, and other areas.

Both site logo and thumbnail are part of SharePoint branding. This post covers how to update the site logo and thumbnail across multiple SharePoint sites within a hub using PowerShell or CLI for Microsoft 365.

I used the script to update site logo and thumbnail for associated sites within a hub site which could be an intranet if there are no valid site logo and thumbnail already on the site.

![Example Screenshot](assets/preview.png)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "URL of the hub site")]
    [string]$HubSiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Local path to the logo file")]
    [string]$LogoPath,

    [Parameter(Mandatory = $true, HelpMessage = "Local path to the thumbnail file")]
    [string]$ThumbnailPath,

    [Parameter(Mandatory = $false, HelpMessage = "Skip file upload if logo/thumbnail already exists")]
    [switch]$SkipIfExists
)

begin {
    $script:sitesProcessed = 0
    $script:sitesFailed = 0

    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = ".\UpdateBranding_$timestamp.log"
    Start-Transcript -Path $transcriptPath

    Write-Host "Validating file paths..." -ForegroundColor Cyan
    if (-not (Test-Path $LogoPath)) {
        throw "Logo file not found: $LogoPath"
    }
    if (-not (Test-Path $ThumbnailPath)) {
        throw "Thumbnail file not found: $ThumbnailPath"
    }

    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365"
    }
}

process {
    Write-Host "Retrieving hub site and associated sites..." -ForegroundColor Cyan
    $hubSiteJson = m365 spo hubsite get --url $HubSiteUrl --withAssociatedSites --output json
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve hub site information"
    }
    $hubSite = $hubSiteJson | ConvertFrom-Json

    if (-not $hubSite.AssociatedSites -or $hubSite.AssociatedSites.Count -eq 0) {
        Write-Host "No associated sites found for hub site: $HubSiteUrl" -ForegroundColor Yellow
        return
    }

    Write-Host "Found $($hubSite.AssociatedSites.Count) associated site(s)" -ForegroundColor Green

    $index = 0
    $total = $hubSite.AssociatedSites.Count

    foreach ($site in $hubSite.AssociatedSites) {
        $index++
        $siteUrl = $site.SiteUrl
        $siteTitle = $site.Title

        Write-Progress -Activity "Updating site branding" `
            -Status "Site $index of $total: $siteTitle" `
            -PercentComplete (($index / $total) * 100) `
            -Id 1

        try {
            $logoFileName = "sitelogo.png"
            $thumbnailFileName = "siteThumbnail.png"
            $logoServerRelativeUrl = ""
            $thumbnailServerRelativeUrl = ""

            if ($SkipIfExists) {
                try {
                    $existingLogoJson = m365 spo file get --webUrl $siteUrl --url "/SiteAssets/$logoFileName" --output json 2>$null
                    if ($LASTEXITCODE -eq 0) {
                        $existingLogo = $existingLogoJson | ConvertFrom-Json
                        $logoServerRelativeUrl = $existingLogo.ServerRelativeUrl
                        Write-Verbose "Logo already exists: $logoServerRelativeUrl"
                    }
                }
                catch {
                    Write-Verbose "Logo does not exist, will upload"
                }
            }

            if (-not $logoServerRelativeUrl) {
                if ($PSCmdlet.ShouldProcess("$siteUrl/SiteAssets/$logoFileName", "Upload logo file")) {
                    Write-Verbose "Uploading logo to $siteUrl"
                    $logoUploadJson = m365 spo file add --webUrl $siteUrl --folder "SiteAssets" --path $LogoPath --fileName $logoFileName --overwrite --output json
                    if ($LASTEXITCODE -ne 0) {
                        throw "Failed to upload logo file"
                    }
                    $logoUpload = $logoUploadJson | ConvertFrom-Json
                    $logoServerRelativeUrl = $logoUpload.ServerRelativeUrl
                }
            }

            if ($SkipIfExists) {
                try {
                    $existingThumbnailJson = m365 spo file get --webUrl $siteUrl --url "/SiteAssets/$thumbnailFileName" --output json 2>$null
                    if ($LASTEXITCODE -eq 0) {
                        $existingThumbnail = $existingThumbnailJson | ConvertFrom-Json
                        $thumbnailServerRelativeUrl = $existingThumbnail.ServerRelativeUrl
                        Write-Verbose "Thumbnail already exists: $thumbnailServerRelativeUrl"
                    }
                }
                catch {
                    Write-Verbose "Thumbnail does not exist, will upload"
                }
            }

            if (-not $thumbnailServerRelativeUrl) {
                if ($PSCmdlet.ShouldProcess("$siteUrl/SiteAssets/$thumbnailFileName", "Upload thumbnail file")) {
                    Write-Verbose "Uploading thumbnail to $siteUrl"
                    $thumbnailUploadJson = m365 spo file add --webUrl $siteUrl --folder "SiteAssets" --path $ThumbnailPath --fileName $thumbnailFileName --overwrite --output json
                    if ($LASTEXITCODE -ne 0) {
                        throw "Failed to upload thumbnail file"
                    }
                    $thumbnailUpload = $thumbnailUploadJson | ConvertFrom-Json
                    $thumbnailServerRelativeUrl = $thumbnailUpload.ServerRelativeUrl
                }
            }

            if ($PSCmdlet.ShouldProcess($siteUrl, "Set site logo and thumbnail")) {
                Write-Verbose "Setting site branding for $siteUrl"
                m365 spo site set --url $siteUrl --siteLogoUrl $logoServerRelativeUrl --siteThumbnailUrl $thumbnailServerRelativeUrl
                if ($LASTEXITCODE -ne 0) {
                    throw "Failed to set site branding"
                }
                Write-Host "Updated branding for: $siteTitle" -ForegroundColor Green
                $script:sitesProcessed++
            }
        }
        catch {
            Write-Warning "Failed to update branding for site '$siteTitle': $($_.Exception.Message)"
            $script:sitesFailed++
        }
    }

    Write-Progress -Activity "Updating site branding" -Id 1 -Completed
}

end {
    Write-Host "`nSummary:" -ForegroundColor Cyan
    Write-Host "Total sites found: $($hubSite.AssociatedSites.Count)" -ForegroundColor White
    Write-Host "Sites processed: $script:sitesProcessed" -ForegroundColor Green
    
    if ($script:sitesFailed -gt 0) {
        Write-Host "Sites failed: $script:sitesFailed" -ForegroundColor Red
    }

    Stop-Transcript
    Write-Host "`nTranscript saved to: $transcriptPath" -ForegroundColor Cyan
}

# Example 1: Update branding for all associated sites
# .\Update-HubSiteBranding.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/hub" -LogoPath "C:\logos\logo.png" -ThumbnailPath "C:\logos\thumbnail.png"

# Example 2: Preview changes without applying them (WhatIf mode)
# .\Update-HubSiteBranding.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/hub" -LogoPath "C:\logos\logo.png" -ThumbnailPath "C:\logos\thumbnail.png" -WhatIf

# Example 3: Skip upload if files already exist
# .\Update-HubSiteBranding.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/hub" -LogoPath "C:\logos\logo.png" -ThumbnailPath "C:\logos\thumbnail.png" -SkipIfExists

# Example 4: Run with verbose output
# .\Update-HubSiteBranding.ps1 -HubSiteUrl "https://contoso.sharepoint.com/sites/hub" -LogoPath "C:\logos\logo.png" -ThumbnailPath "C:\logos\thumbnail.png" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]


# [PnP PowerShell](#tab/pnpps)

```powershell
$AdminCenterURL="https://contoso-admin.sharepoint.com"
$tenantUrl = "https://contoso.sharepoint.com"
$hubSiteUrl = "https://contoso.sharepoint.com"

$logoLocalPath = (Get-Location).Path + '\Logo\logo white 1280x1280.png'
$thumbnailLocalPath = (Get-Location).Path + '\Logo\thumbnail white 1280x1280.png'

Connect-PnPOnline -Url $AdminCenterURL -Interactive
 
$logoLocalPath = (Get-Location).Path + '\Logo\logo white 1280x1280.png'
$thumbnailLocalPath = (Get-Location).Path + '\Logo\thumbnail white 1280x1280.png'
 
$m365Sites = Get-PnPHubSiteChild -Identity $hubSiteUrl

Start-Transcript
 
$m365Sites | ForEach-Object {
    Connect-PnPOnline -Url $_ -Interactive
    $logoFile = '__rectSitelogo__contoso-logo.png'
    $thumbnailFile = 'thumbnail.png'

    $logoRelativePath = $_.replace($tenantUrl, "") + "/SiteAssets/" + $logoFile
    $thumbnailRelativePath = $_.replace($tenantUrl, "") + "/SiteAssets/" + $logoFile

    $logoAslistItem =  Get-PnPFile -Url $logoRelativePath -AsListItem
    $thumbnailAslistItem =  Get-PnPFile -Url $thumbnailRelativePath -AsListItem
 
    if(!$logoAslistItem){
      Add-PnPFile -Path $logoLocalPath -Folder 'SiteAssets' -NewFileName $logoFile | Out-Null
    }
    
    if(!$thumbnailAslistItem){
      Add-PnPFile -Path $thumbnailLocalPath -Folder 'SiteAssets' -NewFileName $thumbnailFile | Out-Null
    }
 
    Set-PnPWebHeader -SiteLogoUrl $logoRelativePath -SiteThumbnailUrl $thumbnailRelativePath
}
Stop-Transcript
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Deletion of sharing links with PowerShell](https://reshmeeauckloo.com/posts/powershell-update-site-header-sitelogo-thumbnail/)

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-branding-sitelogo-thumbnail" aria-hidden="true" />

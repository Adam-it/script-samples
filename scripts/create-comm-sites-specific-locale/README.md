

# Create Communication Sites with a specific primary language

## Summary

Do you want to create a Communication Site in another language? This script will show you how by creating a modern site with the primary language set to a language other than English.

![Example Screenshot](assets/example.png)

> [!Note]
> Once you create a site in the primary language you cannot change it, however you can add support for other languages.

# [PnP PowerShell](#tab/pnpps)

```powershell

$adminUrl = "https://<tenant>-admin.sharepoint.com"
$newSiteUrl = "https://<tenant>.sharepoint.com/sites/Pensaerniaeth" 
$ownerEmail = "<your.name@your.email.com>"

$siteTitle = "Pensaerniaeth"                # Translates to "Architecture" - Bing Translator
$siteTemplate = "SITEPAGEPUBLISHING#0"      # Communication Site Template
$lcid = 1106                                # Welsh
$timeZone = 2                               # London (https://capa.ltd/sp-timezones)

Connect-PnPOnline -Url $adminUrl -NoTelemetry
New-PnPTenantSite -Template $siteTemplate -Title $siteTitle -Url $newSiteUrl `
        -Lcid $lcid -Owner $ownerEmail -TimeZone $timeZone

Write-Host "Script Complete! :)" -ForegroundColor Green

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [SPO Management Shell](#tab/spoms-ps)

```powershell
$adminUrl = "https://<tenant>-admin.sharepoint.com"
$newSiteUrl = "https://<tenant>.sharepoint.com/sites/Pensaerniaeth" 
$ownerEmail = "<your.name@your.email.com>"

$siteTitle = "Pensaerniaeth"                # Translates to "Architecture" - Bing Translator
$siteTemplate = "SITEPAGEPUBLISHING#0"      # Communication Site Template
$lcid = 1106                                # Welsh
$timeZone = 2                               # London
$storageQuota = 1000

Connect-SPOService $adminUrl
New-SPOSite -Template $siteTemplate -Title $siteTitle -Url $newSiteUrl `
        -LocaleId $lcid -Owner $ownerEmail -TimeZoneId $timeZone -StorageQuota $storageQuota

Write-Host "Script Complete! :)" -ForegroundColor Green

```
[!INCLUDE [More about SPO Management Shell](../../docfx/includes/MORE-SPOMS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
  [Parameter(Mandatory, HelpMessage = "The URL of the new Communication Site (e.g., https://contoso.sharepoint.com/sites/MySite)")]
  [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)/sites/[^/]+$')]
  [string]$SiteUrl,

  [Parameter(Mandatory, HelpMessage = "The title of the new Communication Site")]
  [ValidateNotNullOrEmpty()]
  [string]$SiteTitle,

  [Parameter(Mandatory, HelpMessage = "The locale ID (LCID) for the site (e.g., 1106 for Welsh)")]
  [ValidateRange(1, 99999)]
  [int]$Lcid,

  [Parameter(HelpMessage = "Optional description for the site")]
  [string]$Description = "",

  [Parameter(HelpMessage = "Optional site design (Topic, Blank, Showcase)")]
  [ValidateSet('Topic', 'Blank', 'Showcase')]
  [string]$SiteDesign = 'Topic'
)

begin {
  # Initialize transcript with timestamp
  $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
  $transcriptPath = "CreateCommSite_$timestamp.log"
  Start-Transcript -Path $transcriptPath -Append

  Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Starting Communication Site creation with specific locale..." -ForegroundColor Cyan

  # Ensure user is authenticated
  Write-Verbose "Ensuring CLI for Microsoft 365 authentication..."
  m365 login --ensure
  if ($LASTEXITCODE -ne 0) {
    Stop-Transcript
    throw "Failed to authenticate with CLI for Microsoft 365. Please check your credentials."
  }
  Write-Verbose "Authentication successful."

  # Build command arguments
  $cmdArgs = @(
    "spo", "site", "add",
    "--type", "CommunicationSite",
    "--url", $SiteUrl,
    "--title", $SiteTitle,
    "--lcid", $Lcid,
    "--output", "json"
  )

  if ($Description) {
    $cmdArgs += "--description", $Description
  }

  if ($SiteDesign -ne 'Topic') {
    $cmdArgs += "--siteDesign", $SiteDesign
  }
}

process {
  try {
    if ($PSCmdlet.ShouldProcess($SiteUrl, "Create Communication Site with LCID $Lcid")) {
      Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Creating Communication Site..." -ForegroundColor Yellow
      Write-Verbose "Site URL: $SiteUrl"
      Write-Verbose "Title: $SiteTitle"
      Write-Verbose "LCID: $Lcid"
      Write-Verbose "Description: $Description"
      Write-Verbose "Design: $SiteDesign"

      # Execute the command
      $result = m365 @cmdArgs
      
      if ($LASTEXITCODE -ne 0) {
        throw "Failed to create Communication Site. CLI command exited with code $LASTEXITCODE"
      }

      # Parse the result (response is a URL string)
      $createdSiteUrl = ($result | ConvertFrom-Json)
      
      Write-Host "[$(Get-Date -Format 'HH:mm:ss')] ✓ Communication Site created successfully!" -ForegroundColor Green
      Write-Host "  Site URL: $createdSiteUrl" -ForegroundColor Gray
      Write-Host "  LCID: $Lcid" -ForegroundColor Gray
    }
  }
  catch {
    Write-Warning "[$(Get-Date -Format 'HH:mm:ss')] Failed to create site: $_"
    throw
  }
}

end {
  Write-Host "`n[$(Get-Date -Format 'HH:mm:ss')] Script completed." -ForegroundColor Cyan
  Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Gray
  Stop-Transcript
}

# Example usage:
# .\Create-CommSiteSpecificLocale.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Pensaerniaeth" -SiteTitle "Pensaerniaeth" -Lcid 1106

# With WhatIf to preview:
# .\Create-CommSiteSpecificLocale.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Pensaerniaeth" -SiteTitle "Pensaerniaeth" -Lcid 1106 -WhatIf

# With verbose output:
# .\Create-CommSiteSpecificLocale.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Pensaerniaeth" -SiteTitle "Pensaerniaeth" -Lcid 1106 -Verbose

# With optional description and design:
# .\Create-CommSiteSpecificLocale.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Pensaerniaeth" -SiteTitle "Pensaerniaeth" -Lcid 1106 -Description "Welsh language site" -SiteDesign "Showcase"
```
m365 spo site add --type CommunicationSite --url $newSiteUrl --title $siteTitle --lcid $lcid

Write-Host "Script Complete! :)" -ForegroundColor Green

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

To see a list of LCIDs, check out the sample [Generate Markdown Report of LCIDs](../generate-markdown-lcids/README.md) to see the full list

## Source Credit

Article first appeared on [https://www.pkbullock.com/blog/2018/create-communication-sites-with-a-specific-primary-language-using-pnp-powershell/](https://www.pkbullock.com/blog/2018/create-communication-sites-with-a-specific-primary-language-using-pnp-powershell/)

## Contributors

| Author(s) |
|-----------|
| Paul Bullock |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]

<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/create-comm-sites-specific-locale" aria-hidden="true" />

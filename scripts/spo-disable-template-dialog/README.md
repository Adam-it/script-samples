

# Disable Web Templates Gallery First Run Dialog

## Summary

When accessing a newly created site collection in SharePoint Online, you are presented with a dialog to select a web template. This script will disable this dialog using PnP PowerShell or CLI for Microsoft 365.

![Example Screenshot](assets/example.png)


# [PnP PowerShell](#tab/pnpps)

```powershell

#connect to the site collection using one of the many options available in PnP PowerShell
$localConn = Connect-PnPOnline -Url $siteUrl -ClientId $ClientId -CertificateBase64Encoded $CertificateBase64Encoded -Tenant $TenantName -ReturnConnection -erroraction stop
                
$Web = Get-PnPWeb -Includes WebTemplatesGalleryFirstRunEnabled -connection $localConn
$Web.WebTemplatesGalleryFirstRunEnabled = $false
$Web.Update()
Invoke-PnPQuery -connection $localConn        

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, ValueFromPipeline, HelpMessage = "The URL of the SharePoint site to update")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl
)

begin {
    $script:Summary = @{
        Processed = 0
        Failures  = 0
    }
    
    Start-Transcript -Path "disable-template-dialog-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
    
    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365"
    }
    Write-Host "  ✓ Connected successfully" -ForegroundColor Green
}

process {
    Write-Verbose "Processing site: $SiteUrl"
    
    if ($PSCmdlet.ShouldProcess($SiteUrl, 'Disable Web Templates Gallery first-run dialog')) {
        try {
            Write-Host "  Disabling template dialog for: $SiteUrl" -ForegroundColor Yellow
            
            m365 spo web set --url $SiteUrl --WebTemplatesGalleryFirstRunEnabled false
            
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to update site"
            }
            
            Write-Host "    ✓ Template dialog disabled successfully" -ForegroundColor Green
            $script:Summary.Processed++
        }
        catch {
            Write-Warning "Failed to process $SiteUrl : $_"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Host "`nSummary:" -ForegroundColor Cyan
    Write-Host "  Sites processed: $($script:Summary.Processed)" -ForegroundColor Green
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "  Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
    
    Stop-Transcript
}

# Example 1: Disable template dialog for a single site
# .\Disable-TemplateDialog.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x"

# Example 2: Use WhatIf to preview changes
# .\Disable-TemplateDialog.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -WhatIf

# Example 3: Process multiple sites from pipeline
# @("https://contoso.sharepoint.com/sites/site1", "https://contoso.sharepoint.com/sites/site2") | .\Disable-TemplateDialog.ps1

# Example 4: Verbose output
# .\Disable-TemplateDialog.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project-x" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-disable-template-dialog" aria-hidden="true" />

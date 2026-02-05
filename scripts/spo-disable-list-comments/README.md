

# Disable SharePoint List Commenting at list level

## Summary

This sample script shows how to disable commenting feature in SharePoint online lists at list level using PnP PowerShell or CLI for Microsoft 365.

Scenario inspired from this blog post: [Enable/Disable SharePoint Online List Comments using PnP PowerShell](https://ganeshsanapblogs.wordpress.com/2023/03/19/enable-disable-sharepoint-online-list-comments-using-pnp-powershell/)

If you want to enable/disable SharePoint list commenting at tenant level, check this PnP script sample: [Disable SharePoint List Commenting at tenant level](https://pnp.github.io/script-samples/spo-disable-list-comments-tenant/README.html)

![Outupt Screenshot](assets/output.png)

# [PnP PowerShell](#tab/pnpps)

```powershell

# SharePoint online site URL
$siteUrl = "https://contoso.sharepoint.com/sites/SPConnect"

# Display name of SharePoint list
$listName = "Comments List"

# Connect to SharePoint online site
Connect-PnPOnline -Url $siteUrl -Interactive
 
# Disable SharePoint online list comments
Set-PnPList -Identity $listName -DisableCommenting $true

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)')]
    [string]$WebUrl,
    
    [Parameter(Mandatory = $true, HelpMessage = "List title/name")]
    [ValidateNotNullOrEmpty()]
    [string]$ListName,
    
    [Parameter(Mandatory = $false, HelpMessage = "Output path for transcript log")]
    [ValidateScript({ Test-Path -Path $_ -PathType Container })]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyy-MM-dd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "cli-disable-comments-log-$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Yellow
    m365 login --ensure
    
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to authenticate with Microsoft 365"
    }
    
    Write-Host "Connection successful!" -ForegroundColor Green
}

process {
    Write-Host "`nDisabling comments for list: $ListName" -ForegroundColor Yellow
    Write-Verbose "Site URL: $WebUrl"
    
    try {
        if ($PSCmdlet.ShouldProcess($ListName, "Disable commenting")) {
            m365 spo list set --webUrl $WebUrl --title $ListName --disableCommenting true
            
            if ($LASTEXITCODE -ne 0) {
                throw "CLI command failed with exit code $LASTEXITCODE"
            }
            
            Write-Host "Successfully disabled comments for list: $ListName" -ForegroundColor Green
        }
    }
    catch {
        Stop-Transcript
        Write-Error "Failed to disable comments for list '$ListName': $_"
        throw
    }
}

end {
    Stop-Transcript
    
    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Site: $WebUrl" -ForegroundColor Gray
    Write-Host "List: $ListName" -ForegroundColor Gray
    Write-Host "Commenting: Disabled" -ForegroundColor Green
    Write-Host "Transcript: $transcriptPath" -ForegroundColor Gray
    Write-Host "============================`n" -ForegroundColor Cyan
}

# Basic usage
# .\Disable-List-Comments.ps1 -WebUrl "https://contoso.sharepoint.com/sites/SPConnect" -ListName "Comments List"

# Test with WhatIf
# .\Disable-List-Comments.ps1 -WebUrl "https://contoso.sharepoint.com/sites/SPConnect" -ListName "Comments List" -WhatIf

# With verbose logging
# .\Disable-List-Comments.ps1 -WebUrl "https://contoso.sharepoint.com/sites/SPConnect" -ListName "Comments List" -Verbose

# Specify custom output path
# .\Disable-List-Comments.ps1 -WebUrl "https://contoso.sharepoint.com/sites/SPConnect" -ListName "Comments List" -OutputPath "C:\Logs"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/about) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-disable-list-comments" aria-hidden="true" />

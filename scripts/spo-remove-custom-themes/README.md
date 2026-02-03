

# Delete custom color themes from SharePoint

## Summary

Have you been creating a lot of beautiful themes lately and testing them in your dev tenant, but don't want to keep them anymore? If yes, then this PowerShell script is for you.
 
 
# [PnP PowerShell](#tab/pnpps)

```powershell

# SharePoint online admin center URL
$SPOAdmminSite = "https://contoso-admin.sharepoint.com"

$themesToKeep = "Contoso Explorers", "Multicolored theme"

# Connect to SharePoint online admin center
Connect-PnPOnline -Url $SPOAdmminSite -Interactive

# Get all themes from the current tenant
$themes = Get-PnPTenantTheme

$themes = $themes | where {-not ($themesToKeep -contains $_.name)}
$themes | Format-Table name

if ($themes.Count -eq 0) { break }

Read-Host -Prompt "Press Enter to start deleting $($themes.Count) themes (CTRL + C to exit)"
$progress = 0
$total = $themes.Count

foreach ($theme in $themes)
{
  $progress++
  write-host $progress / $total":" $theme.name
  
  # Delete custom color themes from SharePoint
  Remove-PnPTenantTheme -Identity "$($theme.name)"
}

# Disconnect SharePoint online connection
Disconnect-PnPOnline

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [SPO Management Shell](#tab/spoms-ps)

```powershell

# SharePoint online admin center URL
$SPOAdmminSite = "https://contoso-admin.sharepoint.com"

$themesToKeep = "Contoso Explorers", "Multicolored theme"

# Connect to SharePoint online admin center
Connect-SPOService -Url $SPOAdmminSite

# Get all themes from the current tenant
$themes = Get-SPOTheme

$themes = $themes | where {-not ($themesToKeep -contains $_.name)}
$themes | Format-Table name

if ($themes.Count -eq 0) { break }

Read-Host -Prompt "Press Enter to start deleting $($themes.Count) themes (CTRL + C to exit)"
$progress = 0
$total = $themes.Count

foreach ($theme in $themes)
{
  $progress++
  write-host $progress / $total":" $theme.name
  
  # Delete custom color themes from SharePoint
  Remove-SPOTheme -Identity "$($theme.name)"
}

# Disconnect SharePoint online connection
Disconnect-SPOService

```

[!INCLUDE [More about SPO Management Shell](../../docfx/includes/MORE-SPOMS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(HelpMessage = "Array of theme names to keep (all others will be deleted)")]
    [string[]]$ThemesToKeep = @("Contoso Explorers", "Multicolored theme"),
    
    [Parameter(HelpMessage = "Skip confirmation prompts and delete themes without asking")]
    [switch]$Force
)

begin {
    $script:Summary = [PSCustomObject]@{
        TotalFound = 0
        KeptByFilter = 0
        Deleted = 0
        Failed = 0
    }
    
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to verify login status. Please run 'm365 login' manually."
    }
    
    Write-Host "Fetching custom themes from tenant..." -ForegroundColor Cyan
    
    $themesJson = m365 spo theme list --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve themes: $themesJson"
    }
    
    $allThemes = @($themesJson | ConvertFrom-Json)
    $script:Summary.TotalFound = $allThemes.Count
    
    if ($allThemes.Count -eq 0) {
        Write-Host "No custom themes found in tenant." -ForegroundColor Yellow
        return
    }
    
    $script:themesToDelete = $allThemes | Where-Object { -not ($ThemesToKeep -contains $_.name) }
    $script:Summary.KeptByFilter = $allThemes.Count - $script:themesToDelete.Count
    
    if ($script:themesToDelete.Count -eq 0) {
        Write-Host "All $($allThemes.Count) themes are in the keep list. Nothing to delete." -ForegroundColor Yellow
        return
    }
    
    Write-Host "\`nThemes to be deleted ($($script:themesToDelete.Count) of $($allThemes.Count)):" -ForegroundColor Cyan
    $script:themesToDelete | Format-Table -Property name -AutoSize
    
    if (-not $Force -and -not $WhatIfPreference) {
        $confirmation = Read-Host "Press Enter to start deleting $($script:themesToDelete.Count) themes (CTRL + C to cancel)"
    }
}

process {
    if ($script:themesToDelete.Count -eq 0) {
        return
    }
    
    $progress = 0
    $total = $script:themesToDelete.Count
    
    foreach ($theme in $script:themesToDelete) {
        $progress++
        $themeName = $theme.name
        
        if (-not $PSCmdlet.ShouldProcess($themeName, "Delete custom theme")) {
            continue
        }
        
        Write-Host "[$progress/$total] Deleting theme: $themeName" -ForegroundColor Cyan
        
        try {
            $removeResult = m365 spo theme remove --name $themeName --force 2>&1
            
            if ($LASTEXITCODE -eq 0) {
                Write-Host "  ✓ Successfully deleted: $themeName" -ForegroundColor Green
                $script:Summary.Deleted++
            }
            else {
                Write-Warning "Failed to delete theme '$themeName': $removeResult"
                $script:Summary.Failed++
            }
        }
        catch {
            Write-Warning "Exception while deleting theme '$themeName': $($_.Exception.Message)"
            $script:Summary.Failed++
        }
    }
}

end {
    if ($script:Summary.TotalFound -eq 0) {
        return
    }
    
    Write-Host "\`n========================================" -ForegroundColor Gray
    Write-Host "Theme Deletion Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Gray
    Write-Host "Total themes found:      $($script:Summary.TotalFound)" -ForegroundColor White
    Write-Host "Themes kept (filtered):  $($script:Summary.KeptByFilter)" -ForegroundColor Yellow
    Write-Host "Themes deleted:          $($script:Summary.Deleted)" -ForegroundColor Green
    
    if ($script:Summary.Failed -gt 0) {
        Write-Host "Themes failed:           $($script:Summary.Failed)" -ForegroundColor Red
    }
    else {
        Write-Host "Themes failed:           $($script:Summary.Failed)" -ForegroundColor Gray
    }
    
    Write-Host "========================================" -ForegroundColor Gray
    
    if ($script:Summary.Failed -gt 0) {
        Write-Host "\`n⚠️  Some themes failed to delete. Review warnings above." -ForegroundColor Yellow
    }
    elseif ($script:Summary.Deleted -gt 0) {
        Write-Host "\`n✓ All themes deleted successfully!" -ForegroundColor Green
    }
}

# Example 1: Delete all themes except those in default keep list (with confirmation)
# .\\Remove-CustomThemes.ps1

# Example 2: Delete all themes except specific ones (skip confirmation with -Force)
# .\\Remove-CustomThemes.ps1 -ThemesToKeep @("Corporate Blue", "Brand Theme") -Force

# Example 3: Test deletion with WhatIf (shows what would be deleted without actually deleting)
# .\\Remove-CustomThemes.ps1 -WhatIf

# Example 4: Delete ALL custom themes (empty keep list with Force)
# .\\Remove-CustomThemes.ps1 -ThemesToKeep @() -Force

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Leon Armston](https://github.com/LeonArmston)|
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/about) |
| [Adam Wójcik](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-remove-custom-themes" aria-hidden="true" />

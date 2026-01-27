

# Update modern web part properties

## Summary

Script will update web part properties on multiple pages by their Id, Instance Id, or Title using PnP PowerShell or CLI for Microsoft 365

## Implementation

- Open Windows PowerShell ISE
- Edit Script and add details like SharePoint tenant URL, Term groups, and the output directory
- Press run

# [PnP PowerShell](#tab/pnpps)
```powershell

Function Update-CCWebpartProperties() {
    PARAM (
        [Parameter(Mandatory = $true)]
        [string]$SiteURL,

        [Parameter(Mandatory = $false)]
        [string[]]$Pages,

        [Parameter(Mandatory = $true)]
        [string]$WebPartIdentity,

        [Parameter(Mandatory = $false)]
        [string]$PropertyKey,

        [Parameter(Mandatory = $true)]
        [object]$PropertyValue
    )

    Try {
        ## Connect to SharePoint Online site  
        Write-Host "Connect to $($SiteURL)"
        Connect-PnPOnline -URL $SiteURL -UseWebLogin

        # If page parameter is empty, loop through all pages
        if ($Pages.Length -lt 1) {
            $pageItems = Get-PnPListItem -List "Site Pages"
            $Pages = $pageItems | ForEach-Object { $_["FileLeafRef"] }
        }

        $Pages | ForEach-Object {
            try {

                $page = Get-PnPPage -Identity $_
                # Get controls on the page with the identity (id, instance id, or title)
                $controls = $page.Controls | Where-Object { $WebPartIdentity -eq $_.Title -or $WebPartIdentity -eq $_.WebPartId -or $WebPartIdentity -eq $_.InstanceId }    
                Write-Host "Found ($($controls.Length)) web part(s)"

                $controls | ForEach-Object {                        
                    Write-Host "Updating web part, Title: $($_.Title), InstanceId: $($_.InstanceId)"
                    try {
                        $webpartJsonObj = ConvertFrom-Json $_.PropertiesJson
                        if ($PropertyKey) {
                            $webpartJsonObj.$PropertyKey = $PropertyValue
                        }
                        else {
                            $webpartJsonObj = $PropertyValue
                        }

                        $_.PropertiesJson = $webpartJsonObj | ConvertTo-Json
                        Write-Host "Web part properties updated!" -ForegroundColor Green

                    }
                    catch {                           
                        Write-Host "Failed updating web part, Title: $($_.Title), InstanceId: $($_.InstanceId), Error: $($_.Exception)"
                    }
                }

                $null = $page.Save()
                $null = $page.Publish()

                Write-Host "$($_) saved and published." -ForegroundColor Green                    

            }
            catch {
                Write-Host "Failed updating $($page.Title): $($_.Exception)" -ForegroundColor Red
            }
        }

        ## Disconnect the context  
        Disconnect-PnPOnline  
    }

    Catch {
        Write-Host $_.Exception
            
        Break
    }

}

Update-CCWebpartProperties -SiteURL https://contoso.sharepoint.com/sites/test -Pages "PnPSamples" -WebPartIdentity "HelloWorld" -PropertyKey "description" -PropertyValue "Sharing is caring!"

# More examples
<#
# Multi pages
Update-CCWebpartProperties -SiteURL https://contoso.sharepoint.com -Pages "Home","PnPCommunity" -WebPartIdentity "HelloWorld" -PropertyKey "description" -PropertyValue "My web part"

# Update all pages
Update-CCWebpartProperties -SiteURL https://contoso.sharepoint.com -WebPartIdentity "HelloWorld" -PropertyKey "description" -PropertyValue "My web part"

# Update by web part Id
Update-CCWebpartProperties -SiteURL https://contoso.sharepoint.com -WebPartIdentity "6f53d9afa5e347db90e63d6eab04b78c" -PropertyKey "description" -PropertyValue "My web part"

# Update more than one property using object

$link = @{
            title = "Environment"
            url = "$siteAbsoluteUrl/SitePages/Environment.aspx"
            iconName = "DocLibrary"
        }
Update-CCWebpartProperties -SiteURL https://contoso.sharepoint.com -WebPartIdentity "Links" -PropertyValue $link

#>


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "SharePoint site URL where pages are located")]
    [string]$SiteUrl,
    
    [Parameter(HelpMessage = "Specific page names to process (default: all pages)")]
    [string[]]$PageNames,
    
    [Parameter(Mandatory, HelpMessage = "Web part identifier: GUID (WebPartId/InstanceId) or Title")]
    [string]$WebPartIdentity,
    
    [Parameter(HelpMessage = "Single property key to update (omit to replace all properties)")]
    [string]$PropertyKey,
    
    [Parameter(Mandatory, HelpMessage = "Property value (string/number/PSCustomObject)")]
    [object]$PropertyValue
)

begin {
    $script:Summary = @{
        PagesProcessed = 0
        WebPartsUpdated = 0
        Failures = 0
    }
    
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    Start-Transcript -Path "UpdateWebPartProperties-$timestamp.log"
    
    Write-Verbose "Validating login status..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to authenticate with Microsoft 365"
    }
}

process {
    if (-not $PageNames) {
        Write-Verbose "Retrieving all pages from $SiteUrl..."
        try {
            $pagesJson = m365 spo page list --webUrl $SiteUrl --output json
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve pages"
            }
            $pages = $pagesJson | ConvertFrom-Json
            $PageNames = $pages.Name
            Write-Host "Found $($PageNames.Count) pages to process" -ForegroundColor Cyan
        }
        catch {
            Write-Warning "Failed to retrieve pages from site: $_"
            $script:Summary.Failures++
            return
        }
    }
    
    foreach ($pageName in $PageNames) {
        Write-Verbose "Processing page: $pageName"
        
        try {
            $controlsJson = m365 spo page control list --webUrl $SiteUrl --pageName $pageName --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve controls from $pageName"
                $script:Summary.Failures++
                continue
            }
            
            $controls = $controlsJson | ConvertFrom-Json
            
            $matchingControls = $controls | Where-Object {
                $_.id -eq $WebPartIdentity -or 
                $_.title -eq $WebPartIdentity -or 
                $_.controlData.webPartId -eq $WebPartIdentity
            }
            
            if ($matchingControls.Count -eq 0) {
                Write-Verbose "No matching web parts found on $pageName"
                continue
            }
            
            Write-Host "  Found $($matchingControls.Count) matching web part(s) on $pageName" -ForegroundColor Yellow
            
            foreach ($control in $matchingControls) {
                if ($PSCmdlet.ShouldProcess("$pageName - $($control.title)", "Update web part properties")) {
                    try {
                        $currentProps = $control.controlData.webPartData.properties
                        
                        if ($PropertyKey) {
                            $currentProps.$PropertyKey = $PropertyValue
                        } else {
                            $currentProps = $PropertyValue
                        }
                        
                        $propsJson = ($currentProps | ConvertTo-Json -Depth 10 -Compress)
                        
                        Write-Verbose "  Updating control $($control.id) on $pageName..."
                        m365 spo page control set --webUrl $SiteUrl --pageName $pageName --id $control.id --webPartProperties $propsJson
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to update control"
                        }
                        
                        Write-Host "    Updated: $($control.title)" -ForegroundColor Green
                        $script:Summary.WebPartsUpdated++
                    }
                    catch {
                        Write-Warning "Failed to update $($control.title) on $pageName: $_"
                        $script:Summary.Failures++
                        continue
                    }
                }
            }
            
            if ($PSCmdlet.ShouldProcess($pageName, "Publish page")) {
                Write-Verbose "  Publishing $pageName..."
                m365 spo page set --webUrl $SiteUrl --name $pageName --publish
                if ($LASTEXITCODE -eq 0) {
                    $script:Summary.PagesProcessed++
                    Write-Host "  Published: $pageName" -ForegroundColor Green
                } else {
                    Write-Warning "Failed to publish $pageName"
                }
            }
        }
        catch {
            Write-Warning "Error processing $pageName: $_"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Host "`nSummary:" -ForegroundColor Cyan
    Write-Host "  Pages processed: $($script:Summary.PagesProcessed)" -ForegroundColor Green
    Write-Host "  Web parts updated: $($script:Summary.WebPartsUpdated)" -ForegroundColor Green
    if ($script:Summary.Failures -gt 0) {
        Write-Host "  Failures: $($script:Summary.Failures)" -ForegroundColor Red
    }
    Stop-Transcript
}

# Example 1: Update single property on specific page
# .\Update-WebPartProperties.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/test" -PageNames "Home.aspx" -WebPartIdentity "HelloWorld" -PropertyKey "description" -PropertyValue "New description"

# Example 2: Update all pages with WhatIf
# .\Update-WebPartProperties.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/test" -WebPartIdentity "HelloWorld" -PropertyKey "title" -PropertyValue "Updated Title" -WhatIf

# Example 3: Update by web part ID (GUID)
# .\Update-WebPartProperties.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/test" -WebPartIdentity "6f53d9af-a5e3-47db-90e6-3d6eab04b78c" -PropertyKey "description" -PropertyValue "Updated via ID"

# Example 4: Replace all properties with object
# $newProps = @{ title = "Links"; description = "Quick links"; iconName = "DocLibrary" }
# .\Update-WebPartProperties.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/test" -WebPartIdentity "Links" -PropertyValue $newProps
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Ramin Ahmadi](https://github.com/ahmadiramin) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-modern-webpart-properties" aria-hidden="true" />

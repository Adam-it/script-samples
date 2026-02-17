# Updates a SharePoint Online lookup column to reference a lookup list

## Summary

This PowerShell script enhances SharePoint Online list management by updating a lookup column in a list to reference a lookup list. It modifies the lookup column's schema XML to set or update the List, WebId, ShowField attributes. The script allows users to specify the primary display field (via ShowFieldName, e.g., Title, ID). Available in both PnP PowerShell and CLI for Microsoft 365 v11.4.0+.

![Example Screenshot](assets/LookupField.png)

# [PnP PowerShell](#tab/pnpps)

```powershell
<#
.SYNOPSIS
    Updates a SharePoint Online lookup column to reference a lookup list
.DESCRIPTION
    This script uses PnP PowerShell to update the schema XML of a lookup column in a source list to point to a recreated lookup list by setting or updating the List, WebId, ShowField. The ShowField attribute sets the primary display field.
.PARAMETER SiteUrl
    The URL of the SharePoint Online site (e.g., https://contoso.sharepoint.com/sites/yoursite).
.PARAMETER TargetListName
    The name of the list containing the lookup column.
.PARAMETER LookupListName
    The name of the lookup list.
.PARAMETER LookupColumnName
    The internal name of the lookup column in the list.
.PARAMETER ShowFieldName
    The internal name of the primary field in the lookup list to display (e.g., Title, ID). Defaults to Title.

.EXAMPLE
    .\Update-LookupColumn.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/yoursite" -TargetListName "targetList" -LookupListName "LookupList" -LookupColumnName "LookupColumn" -ShowFieldName "ID"
    Updates the lookup column "LookupColumn" in "targetList" to reference "LookupList"
.NOTES
    - Ensure the PnP PowerShell module is installed: Install-Module -Name PnP.PowerShell.
    - The lookup list must have the fields specified in ShowFieldName.

.LINK
    https://pnp.github.io/script-samples
#>

[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "The URL of the SharePoint Online site.")]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "The name of the source list containing the lookup column.")]
    [string]$TargetListName,

    [Parameter(Mandatory = $true, HelpMessage = "The name of the lookup list.")]
    [string]$LookupListName,

    [Parameter(Mandatory = $true, HelpMessage = "The internal name of the lookup column in the source list.")]
    [string]$LookupColumnName,

    [Parameter(Mandatory = $false, HelpMessage = "The internal name of the primary field in the lookup list to display (e.g., Title, ID).")]
    [string]$ShowFieldName = "Title"
)

try {
    # Connect to the SharePoint site
    $ClientId = "<your-client-id>" # Replace with your Microsoft Entra ID (Azure AD) app client ID
    $Tenant= "<your-tenant-name>" #contoso.sharepoint.com

    Write-Host "Connecting to SharePoint site: $SiteUrl" -ForegroundColor Cyan
    Connect-PnPOnline -Url $SiteUrl -ClientId $ClientId -Tenant $Tenant  -Interactive -ErrorAction Stop
    Write-Host "Connected successfully." -ForegroundColor Green
    # Get the recreated lookup list
    $lookupList = Get-PnPList -Identity $LookupListName -ErrorAction Stop
    if ($null -eq $lookupList) {
        throw "Lookup list '$LookupListName' not found."
    }

    # Get the source list
    $targetList = Get-PnPList -Identity $TargetListName -Includes Fields -ErrorAction Stop
    if ($null -eq $targetList) {
        throw "Source list '$TargetListName' not found."
    }

    # Get the lookup field
    $lookupField = Get-PnPField -List $targetList.Title -Identity $LookupColumnName -ErrorAction Stop
    if ($null -eq $lookupField) {
        throw "Lookup column '$LookupColumnName' not found in list '$TargetListName'."
    }

    # Validate ShowFieldName
    $showField = Get-PnPField -List $lookupList.Title -Identity $ShowFieldName -ErrorAction Stop
    if ($null -eq $showField) {
        throw "Field '$ShowFieldName' not found in lookup list '$LookupListName'."
    }

    # Get the lookup list's GUID and web ID
    $lookupListId = $lookupList.Id
    $web = Get-PnPWeb -ErrorAction Stop
    $webId = $web.Id

    # Get the current schema XML
    $schemaXml = $lookupField.SchemaXml

    # Update or add List attribute
    $listPattern = 'List="{[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}}"'
    if ($schemaXml -match $listPattern) {
        $schemaXml = $schemaXml -replace $listPattern, "List=`"$lookupListId`""
        Write-Host "Updated existing List attribute with new GUID: $lookupListId" -ForegroundColor Green
    }
    elseif ($schemaXml -match 'List="[^"]*"') {
        $schemaXml = $schemaXml -replace 'List="[^"]*"', "List=`"$lookupListId`""
        Write-Host "Replaced invalid List attribute with new GUID: $lookupListId" -ForegroundColor Green
    }
    else {
        $schemaXml = $schemaXml -replace '/>', " List=`"$lookupListId`"/>"
        Write-Host "Added missing List attribute with GUID: $lookupListId" -ForegroundColor Green
    }

    # Update or add WebId attribute
    $webIdPattern = 'WebId="{[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}}"'
    if ($schemaXml -match $webIdPattern) {
        $schemaXml = $schemaXml -replace $webIdPattern, "WebId=`"$webId`""
        Write-Host "Updated existing WebId attribute with new GUID: $webId" -ForegroundColor Green
    }
    elseif ($schemaXml -match 'WebId="[^"]*"') {
        $schemaXml = $schemaXml -replace 'WebId="[^"]*"', "WebId=`"$webId`""
        Write-Host "Replaced invalid WebId attribute with new GUID: $webId" -ForegroundColor Green
    }
    else {
        $schemaXml = $schemaXml -replace '/>', " WebId=`"$webId`"/>"
        Write-Host "Added missing WebId attribute with GUID: $webId" -ForegroundColor Green
    }

    # Update or add ShowField attribute
    $showFieldPattern = 'ShowField="[^"]*"'
    if ($schemaXml -match $showFieldPattern) {
        $schemaXml = $schemaXml -replace $showFieldPattern, "ShowField=`"$ShowFieldName`""
        Write-Host "Updated ShowField attribute to: $ShowFieldName" -ForegroundColor Green
    }
    else {
        $schemaXml = $schemaXml -replace '/>', " ShowField=`"$ShowFieldName`"/>"
        Write-Host "Added missing ShowField attribute: $ShowFieldName" -ForegroundColor Green
    }


    # Apply the updated schema
    Set-PnPField -List $targetList -Identity $LookupColumnName -Values @{SchemaXml = $schemaXml } -ErrorAction Stop
    Write-Host "Updated lookup column schema for '$LookupColumnName'." -ForegroundColor Green

}
catch {
    Write-Host "Error: $($_.Exception.Message)" -ForegroundColor Red
    Write-Host "Stack Trace: $($_.ScriptStackTrace)" -ForegroundColor Red
}
finally {
    # Disconnect from the site
    Disconnect-PnPOnline -ErrorAction SilentlyContinue
    Write-Host "Disconnected from SharePoint site." -ForegroundColor Cyan
}
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage="SharePoint site URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory, HelpMessage="Target list containing lookup column")]
    [string]$TargetListName,
    
    [Parameter(Mandatory, HelpMessage="Lookup list to reference")]
    [string]$LookupListName,
    
    [Parameter(Mandatory, HelpMessage="Internal name of lookup column")]
    [string]$LookupColumnName,
    
    [Parameter(HelpMessage="Field in lookup list to display (default: Title)")]
    [string]$ShowFieldName = "Title"
)

begin {
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    
    Write-Host "Updating lookup field schema..." -ForegroundColor Cyan
    Write-Host "Site: $SiteUrl" -ForegroundColor Cyan
    Write-Host "Target List: $TargetListName" -ForegroundColor Cyan
    Write-Host "Lookup List: $LookupListName" -ForegroundColor Cyan
    Write-Host "Lookup Column: $LookupColumnName" -ForegroundColor Cyan
    Write-Host "Show Field: $ShowFieldName`n" -ForegroundColor Cyan
    
    try {
        Write-Verbose "Getting web GUID..."
        $webJson = m365 spo web get --url $SiteUrl --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to get web details for site: $SiteUrl"
        }
        $web = $webJson | ConvertFrom-Json
        $webId = $web.Id
        Write-Verbose "Web GUID: $webId"
        
        Write-Verbose "Getting lookup list details..."
        $lookupListJson = m365 spo list get --webUrl $SiteUrl --title $LookupListName --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Lookup list '$LookupListName' not found"
        }
        $lookupList = $lookupListJson | ConvertFrom-Json
        $lookupListId = $lookupList.Id
        Write-Verbose "Lookup List GUID: $lookupListId"
        
        Write-Verbose "Getting target list details..."
        $targetListJson = m365 spo list get --webUrl $SiteUrl --title $TargetListName --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Target list '$TargetListName' not found"
        }
        $targetList = $targetListJson | ConvertFrom-Json
        Write-Verbose "Target List GUID: $($targetList.Id)"
        
        Write-Verbose "Validating ShowFieldName exists in lookup list..."
        $showFieldJson = m365 spo field get --webUrl $SiteUrl --listTitle $LookupListName --internalName $ShowFieldName --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Field '$ShowFieldName' not found in lookup list '$LookupListName'"
        }
        Write-Verbose "ShowField '$ShowFieldName' validated"
        
        Write-Verbose "Getting lookup field details..."
        $lookupFieldJson = m365 spo field get --webUrl $SiteUrl --listTitle $TargetListName --internalName $LookupColumnName --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Lookup column '$LookupColumnName' not found in list '$TargetListName'"
        }
        $lookupField = $lookupFieldJson | ConvertFrom-Json
        $schemaXml = $lookupField.SchemaXml
        Write-Verbose "Current SchemaXml retrieved"
        
        $script:UpdateDetails = @{
            WebId = $webId
            LookupListId = $lookupListId
            SchemaXml = $schemaXml
            ListUpdated = $false
            WebIdUpdated = $false
            ShowFieldUpdated = $false
        }
    }
    catch {
        throw "Initialization failed: $($_.Exception.Message)"
    }
}

process {
    try {
        $schemaXml = $script:UpdateDetails.SchemaXml
        
        Write-Verbose "Updating List attribute in SchemaXml..."
        $listPattern = 'List="\{[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}\}"'
        if ($schemaXml -match $listPattern) {
            $schemaXml = $schemaXml -replace $listPattern, "List=`\"$($script:UpdateDetails.LookupListId)`\""
            Write-Host "  ✓ Updated existing List attribute with new GUID: $($script:UpdateDetails.LookupListId)" -ForegroundColor Green
            $script:UpdateDetails.ListUpdated = $true
        }
        elseif ($schemaXml -match 'List="[^"]*"') {
            $schemaXml = $schemaXml -replace 'List="[^"]*"', "List=`\"$($script:UpdateDetails.LookupListId)`\""
            Write-Host "  ✓ Replaced invalid List attribute with new GUID: $($script:UpdateDetails.LookupListId)" -ForegroundColor Green
            $script:UpdateDetails.ListUpdated = $true
        }
        else {
            $schemaXml = $schemaXml -replace '/>', " List=`\"$($script:UpdateDetails.LookupListId)`\"/>"
            Write-Host "  ✓ Added missing List attribute with GUID: $($script:UpdateDetails.LookupListId)" -ForegroundColor Green
            $script:UpdateDetails.ListUpdated = $true
        }
        
        Write-Verbose "Updating WebId attribute in SchemaXml..."
        $webIdPattern = 'WebId="\{[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}\}"'
        if ($schemaXml -match $webIdPattern) {
            $schemaXml = $schemaXml -replace $webIdPattern, "WebId=`\"$($script:UpdateDetails.WebId)`\""
            Write-Host "  ✓ Updated existing WebId attribute with new GUID: $($script:UpdateDetails.WebId)" -ForegroundColor Green
            $script:UpdateDetails.WebIdUpdated = $true
        }
        elseif ($schemaXml -match 'WebId="[^"]*"') {
            $schemaXml = $schemaXml -replace 'WebId="[^"]*"', "WebId=`\"$($script:UpdateDetails.WebId)`\""
            Write-Host "  ✓ Replaced invalid WebId attribute with new GUID: $($script:UpdateDetails.WebId)" -ForegroundColor Green
            $script:UpdateDetails.WebIdUpdated = $true
        }
        else {
            $schemaXml = $schemaXml -replace '/>', " WebId=`\"$($script:UpdateDetails.WebId)`\"/>"
            Write-Host "  ✓ Added missing WebId attribute with GUID: $($script:UpdateDetails.WebId)" -ForegroundColor Green
            $script:UpdateDetails.WebIdUpdated = $true
        }
        
        Write-Verbose "Updating ShowField attribute in SchemaXml..."
        $showFieldPattern = 'ShowField="[^"]*"'
        if ($schemaXml -match $showFieldPattern) {
            $schemaXml = $schemaXml -replace $showFieldPattern, "ShowField=`\"$ShowFieldName`\""
            Write-Host "  ✓ Updated ShowField attribute to: $ShowFieldName" -ForegroundColor Green
            $script:UpdateDetails.ShowFieldUpdated = $true
        }
        else {
            $schemaXml = $schemaXml -replace '/>', " ShowField=`\"$ShowFieldName`\"/>"
            Write-Host "  ✓ Added missing ShowField attribute: $ShowFieldName" -ForegroundColor Green
            $script:UpdateDetails.ShowFieldUpdated = $true
        }
        
        if ($PSCmdlet.ShouldProcess("$TargetListName.$LookupColumnName", "Update lookup field schema")) {
            Write-Verbose "Applying updated SchemaXml to field..."
            m365 spo field set --webUrl $SiteUrl --listTitle $TargetListName --internalName $LookupColumnName --SchemaXml $schemaXml
            
            if ($LASTEXITCODE -eq 0) {
                Write-Host "`n✓ Successfully updated lookup field '$LookupColumnName'" -ForegroundColor Green
            }
            else {
                throw "Failed to update lookup field schema"
            }
        }
    }
    catch {
        throw "Schema update failed: $($_.Exception.Message)"
    }
}

end {
    Write-Host "`n=== Update Summary ===" -ForegroundColor Cyan
    Write-Host "List attribute updated: $($script:UpdateDetails.ListUpdated)" -ForegroundColor White
    Write-Host "WebId attribute updated: $($script:UpdateDetails.WebIdUpdated)" -ForegroundColor White
    Write-Host "ShowField attribute updated: $($script:UpdateDetails.ShowFieldUpdated)" -ForegroundColor White
    Write-Host "`nLookup field schema updated successfully" -ForegroundColor Green
}

# .\Update-LookupField.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/site" -TargetListName "Orders" -LookupListName "Products" -LookupColumnName "Product"

# .\Update-LookupField.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/site" -TargetListName "Orders" -LookupListName "Products" -LookupColumnName "Product" -ShowFieldName "ID" -WhatIf

# .\Update-LookupField.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/site" -TargetListName "Orders" -LookupListName "Products" -LookupColumnName "Product" -Verbose

# .\Update-LookupField.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/site" -TargetListName "Orders" -LookupListName "Products" -LookupColumnName "Product" -ShowFieldName "ProductCode"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

---

## Contributors

| Author(s) |
| --------- |

| [Harminder Singh](https://github.com/harmindersethi) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-lookup-filed" aria-hidden="true" />

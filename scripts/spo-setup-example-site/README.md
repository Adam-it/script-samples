

# Setup example site

## Summary

This PnP PowerShell or CLI for Microsoft 365 script is a good starting point for a setup script to create site with some assets like columns, content types, lists, navigation etc. The given example:
 - creates a site,
 - adds a site column and a content type,
 - adds list and modifies it's settings (add a content type to it and makes it hidden),
 - adds a document library with a custom column and some folder,
 - modifies the all items view of the document library,
 - modifies the site navigation links
 

# [PnP PowerShell](#tab/pnpps)
```powershell
###### Declare and Initialize Variables ######  

Write-host 'setup script example'
## Connect to SharePoint Online site  
Connect-PnPOnline -Url $Url -Interactive


Write-host 'create setup site'
$siteRelativeUrl = 'sites/setupTestSitePNP'
$tenantUrl = 'https://<tenant>.sharepoint.com'
$siteUrl = "$tenantUrl/$siteRelativeUrl"
$siteTitle = 'setup test site PNP'
$siteType = 'CommunicationSite'

$site = Get-PnPTenantSite -Identity  $siteUrl

if ($null -eq $site) {
  Write-host 'setup site does not exist, I will create it'
  New-PnPSite -Type $siteType -Title $siteTitle -Url $siteUrl
}
else {
  Write-host 'setup site already exists'
}

## Disconnect the context  
Disconnect-PnPOnline  
## Connect to SharePoint Online site  
Connect-PnPOnline -Url $siteUrl -Interactive

Write-host 'add site column'
$fieldName = 'Sample Text Column PNP'
$field = Get-PnPField -Identity $fieldName
if ($null -eq $field) {
  Write-host 'sample site column does not exist, I will create it'
  $fieldXml = "<Field ID='{13AFECC0-2454-41F3-85E6-E194458C861C}' Type='Text' Name='SampleTextColumnPNP' DisplayName='Sample Text Column PNP' Indexed='FALSE' Group='Sample Columns PNP' Required='FALSE' SourceID='{4f118c69-66e0-497c-96ff-d7855ce0713d}' StaticName='SampleTextColumnPNP' FromBaseType='TRUE' ></Field>"
  $field = Add-PnPFieldFromXml -FieldXml $fieldXml 
}
else {
  Write-host 'sample site column already exists'
}

Write-host 'add site content type'
$contentTypeName = 'sample content type PNP'
$contentTypeGroup = 'sample content type group PNP'
$parentId = '0x01007926A45D687BA842B947286090B8F67D' # list item content type
$contentType = Get-PnPContentType -Identity $contentTypeName

if ($null -eq $contentType) {
  Write-host 'sample site content type does not exist, I will create it'
  $ct = Get-PnPContentType -Identity Item
  $contentType = Add-PnPContentType -Name $contentTypeName  -Group $contentTypeGroup -ParentContentType $ct 
  $contentType = Get-PnPContentType -Identity $contentTypeName

}
else {
  Write-host 'sample site content type already exists'
}


Write-host 'add field to content type'
$fieldId = $field.Id
$contentTypeId = $contentType.StringId
Add-PnPFieldToContentType -Field $fieldId -ContentType $contentTypeId

Write-host 'create generic list'
$listName = 'setup test list PNP'
$list = Get-PnPList -Identity $listName
if ($null -eq $list) {
  Write-host 'sample generic list does not exist, I will create it'
  $list = New-PnPList -Title $listName -Template GenericList
}
else {
  Write-host 'sample generic list already exists'
}

Write-host 'modify list settings to allow content types'
Set-PnPList -Identity $list -EnableContentTypes $true


Write-host 'add content type to list'
$contentTypeAddedToList = Add-PnPContentTypeToList -List $list -ContentType $contentTypeId -DefaultContentType


Write-host 'make list hidden'
Set-PnPList -Identity $list -Hidden $true

Write-host 'create document lib'
$libName = 'setup test lib PNP'
$lib = Get-PnPList -Identity $libName

if ($null -eq $lib) {
  Write-host 'sample document lib does not exist, I will create it'
  $lib = New-PnPList -Title $libName -Template DocumentLibrary
}
else {
  Write-host 'sample document lib already exists'
}


Write-host 'add sample column'
$columnName = 'Sample Text Column PNP'
$column = Get-PnPField -List $libName -Identity $columnName

if ($null -eq $column) {
  Write-host 'sample column in lib does not exist, I will create it'
  $columnXml = "<Field ID='{AC827B0C-8B45-4B4F-927B-CDDC4FEEE79E}' Type='Text' Name='SampleTextColumnPNP' DisplayName='Sample Text Column PNP' Required='FALSE' SourceID='http://schemas.microsoft.com/sharepoint/v3' StaticName='SampleTextColumnPNP' FromBaseType='TRUE' />"
  $column = Add-PnPFieldFromXml -List $libName -FieldXml $columnXml
  
}
else {
  Write-host 'sample column in lib already exists'
}


Write-host 'add sample folder'
$folderName = 'sample Folder PNP'
$folder = Get-PnPFolder -List $libName 

if ($null -eq $folder) {
  Write-host 'sample folder in lib does not exist, I will create it'
  $folder = Add-PnPFolder -Name $folderName -Folder $libName
  
}
else {
  Write-host 'sample folder in lib already exists'
}

Write-host 'modify list view'
$views = Get-PnPView -List $list

$viewName = $views[0].Title # all items view
Set-PnPView -List $list -Identity $viewName -Fields $columnName

Write-host 'modify site navigation'
$currentNavigation = Get-PnPNavigationNode -Location QuickLaunch

Write-host 'clearing old navigation links'
foreach ($navigationItem in $currentNavigation) {
    Remove-PnPNavigationNode -identity $navigationItem.Id   -Location QuickLaunch -Force 
  
}
Write-host 'adding new navigation'
$nodeAddedResponse = Add-PnPNavigationNode -Title "Sample Document Library PNP" -Url "/$siteRelativeUrl/$libName/Forms/AllItems.aspx" -Location "QuickLaunch"
$nodeAddedResponse = Add-PnPNavigationNode -Title "Hidden Sample List PNP" -Url "/$siteRelativeUrl/Lists/$listName/AllItems.aspx" -Location "QuickLaunch"

 
## Disconnect the context  
Disconnect-PnPOnline  
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(HelpMessage = "Site relative URL (e.g., 'sites/setup-example')")]
    [string]$SiteRelativeUrl = "sites/setupTestCLI",
    
    [Parameter(HelpMessage = "Site title")]
    [string]$SiteTitle = "Setup Test Site CLI"
)

begin {
    $script:Summary = @{
        SiteColumn = 0
        ContentType = 0
        List = 0
        Library = 0
        Folder = 0
        Navigation = 0
        Failures = 0
    }
    
    $transcriptPath = "$((Get-Location).Path)/SetupExampleSite-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Setup Example Site - CLI for Microsoft 365" -ForegroundColor Cyan
    Write-Host "=============================================" -ForegroundColor Cyan
    Write-Host ""
    
    Write-Verbose "Ensuring Microsoft 365 login..."
    m365 login --ensure | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure Microsoft 365 login. Please run 'm365 login' first."
    }
    Write-Host "✓ Authenticated successfully" -ForegroundColor Green
    Write-Host ""
    
    Write-Verbose "Retrieving tenant URL..."
    $spoGetResult = m365 spo get --output json
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve tenant URL. Error: $spoGetResult"
    }
    $spoContext = $spoGetResult | ConvertFrom-Json
    $tenantUrl = $spoContext.SpoUrl
    Write-Verbose "Tenant URL: $tenantUrl"
    
    $script:SiteUrl = "$tenantUrl/$SiteRelativeUrl"
    Write-Host "Target Site URL: $script:SiteUrl" -ForegroundColor White
    Write-Host ""
}

process {
    
    Write-Verbose "Checking if site exists..."
    $siteCheckResult = m365 spo site get --url $script:SiteUrl --output json 2>&1
    
    if ($LASTEXITCODE -ne 0) {
        Write-Host "Site does not exist. Creating communication site..." -ForegroundColor Yellow
        
        if ($PSCmdlet.ShouldProcess($script:SiteUrl, 'Create communication site')) {
            try {
                $siteResult = m365 spo site add --type CommunicationSite --url $script:SiteUrl --title $SiteTitle --output json
                if ($LASTEXITCODE -ne 0) {
                    throw "Failed to create site: $siteResult"
                }
                Write-Host "✓ Site created successfully" -ForegroundColor Green
            }
            catch {
                Write-Warning "Failed to create site: $_"
                $script:Summary.Failures++
                throw
            }
        }
    }
    else {
        Write-Host "✓ Site already exists" -ForegroundColor Green
    }
    
    Write-Host ""
    Write-Host "Creating SharePoint assets..." -ForegroundColor Cyan
    Write-Host ""
    
    $fieldName = "Sample Text Column CLI"
    $fieldInternalName = "SampleTextColumnCLI"
    $fieldId = "{13AFECC0-2454-41F3-85E6-E194458C861C}"
    $fieldGroup = "Sample Columns CLI"
    
    Write-Verbose "Checking if site column exists..."
    $fieldsJson = m365 spo field list --webUrl $script:SiteUrl --output json
    if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve fields: $fieldsJson"
        $script:Summary.Failures++
    }
    else {
        $fields = @($fieldsJson | ConvertFrom-Json)
        $existingField = $fields | Where-Object { $_.Title -eq $fieldName }
        
        if ($null -eq $existingField) {
            Write-Host "Creating site column '$fieldName'..." -ForegroundColor Yellow
            
            if ($PSCmdlet.ShouldProcess($fieldName, 'Create site column')) {
                try {
                    $fieldXml = "<Field Type='Text' DisplayName='$fieldName' Required='FALSE' EnforceUniqueValues='FALSE' Indexed='FALSE' Group='$fieldGroup' ID='$fieldId' StaticName='$fieldInternalName' Name='$fieldInternalName' FromBaseType='TRUE'></Field>"
                    $fieldResult = m365 spo field add --webUrl $script:SiteUrl --xml $fieldXml --output json
                    if ($LASTEXITCODE -ne 0) {
                        throw "CLI error: $fieldResult"
                    }
                    $script:Summary.SiteColumn++
                    Write-Host "✓ Site column created successfully" -ForegroundColor Green
                }
                catch {
                    Write-Warning "Failed to create site column: $_"
                    $script:Summary.Failures++
                }
            }
            else {
                $script:Summary.SiteColumn++
            }
        }
        else {
            Write-Host "✓ Site column '$fieldName' already exists" -ForegroundColor Gray
        }
    }
    
    $contentTypeName = "sample content type CLI"
    $contentTypeGroup = "sample content type group CLI"
    $parentContentTypeId = "0x01007926A45D687BA842B947286090B8F67D"
    
    Write-Verbose "Checking if content type exists..."
    $contentTypesJson = m365 spo contenttype list --webUrl $script:SiteUrl --output json
    if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to retrieve content types: $contentTypesJson"
        $script:Summary.Failures++
    }
    else {
        $contentTypes = @($contentTypesJson | ConvertFrom-Json)
        $existingContentType = $contentTypes | Where-Object { $_.Name -eq $contentTypeName }
        
        if ($null -eq $existingContentType) {
            Write-Host "Creating content type '$contentTypeName'..." -ForegroundColor Yellow
            
            if ($PSCmdlet.ShouldProcess($contentTypeName, 'Create content type')) {
                try {
                    $contentTypeResult = m365 spo contenttype add --webUrl $script:SiteUrl --name $contentTypeName --id $parentContentTypeId --group $contentTypeGroup --output json
                    if ($LASTEXITCODE -ne 0) {
                        throw "CLI error: $contentTypeResult"
                    }
                    $createdContentType = $contentTypeResult | ConvertFrom-Json
                    $script:Summary.ContentType++
                    Write-Host "✓ Content type created successfully" -ForegroundColor Green
                    
                    Write-Verbose "Adding field to content type..."
                    if ($PSCmdlet.ShouldProcess("$contentTypeName/$fieldName", 'Add field to content type')) {
                        try {
                            $fieldsJson = m365 spo field list --webUrl $script:SiteUrl --output json
                            $fields = @($fieldsJson | ConvertFrom-Json)
                            $field = $fields | Where-Object { $_.Title -eq $fieldName }
                            
                            if ($null -ne $field) {
                                $fieldSetResult = m365 spo contenttype field set --webUrl $script:SiteUrl --contentTypeId $createdContentType.StringId --fieldId $field.Id --output json 2>&1
                                if ($LASTEXITCODE -ne 0) {
                                    Write-Warning "Failed to add field to content type: $fieldSetResult"
                                    $script:Summary.Failures++
                                }
                                else {
                                    Write-Host "  ✓ Field added to content type" -ForegroundColor Green
                                }
                            }
                        }
                        catch {
                            Write-Warning "Failed to add field to content type: $_"
                            $script:Summary.Failures++
                        }
                    }
                }
                catch {
                    Write-Warning "Failed to create content type: $_"
                    $script:Summary.Failures++
                }
            }
            else {
                $script:Summary.ContentType++
            }
        }
        else {
            Write-Host "✓ Content type '$contentTypeName' already exists" -ForegroundColor Gray
            $existingContentType = $existingContentType
        }
    }
    
    $listName = "Sample List CLI"
    
    Write-Verbose "Checking if list exists..."
    $listCheckResult = m365 spo list get --webUrl $script:SiteUrl --title $listName --output json 2>&1
    
    if ($LASTEXITCODE -ne 0) {
        Write-Host "Creating generic list '$listName'..." -ForegroundColor Yellow
        
        if ($PSCmdlet.ShouldProcess($listName, 'Create generic list')) {
            try {
                $listResult = m365 spo list add --webUrl $script:SiteUrl --title $listName --baseTemplate GenericList --output json
                if ($LASTEXITCODE -ne 0) {
                    throw "CLI error: $listResult"
                }
                $script:Summary.List++
                Write-Host "✓ List created successfully" -ForegroundColor Green
                
                Write-Verbose "Enabling content types on list..."
                if ($PSCmdlet.ShouldProcess($listName, 'Enable content types')) {
                    $listSetResult = m365 spo list set --webUrl $script:SiteUrl --title $listName --enableContentTypes true --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to enable content types: $listSetResult"
                        $script:Summary.Failures++
                    }
                    else {
                        Write-Host "  ✓ Content types enabled" -ForegroundColor Green
                    }
                }
                
                Write-Verbose "Adding content type to list..."
                $contentTypesJson = m365 spo contenttype list --webUrl $script:SiteUrl --output json
                $contentTypes = @($contentTypesJson | ConvertFrom-Json)
                $contentType = $contentTypes | Where-Object { $_.Name -eq $contentTypeName }
                
                if ($null -ne $contentType -and $PSCmdlet.ShouldProcess("$listName/$contentTypeName", 'Add content type to list')) {
                    $ctAddResult = m365 spo list contenttype add --webUrl $script:SiteUrl --listTitle $listName --contentTypeId $contentType.StringId --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to add content type to list: $ctAddResult"
                        $script:Summary.Failures++
                    }
                    else {
                        Write-Host "  ✓ Content type added to list" -ForegroundColor Green
                    }
                }
                
                Write-Verbose "Hiding list..."
                if ($PSCmdlet.ShouldProcess($listName, 'Hide list')) {
                    $hideResult = m365 spo list set --webUrl $script:SiteUrl --title $listName --hidden true --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to hide list: $hideResult"
                        $script:Summary.Failures++
                    }
                    else {
                        Write-Host "  ✓ List hidden" -ForegroundColor Green
                    }
                }
            }
            catch {
                Write-Warning "Failed to create list: $_"
                $script:Summary.Failures++
            }
        }
        else {
            $script:Summary.List++
        }
    }
    else {
        Write-Host "✓ List '$listName' already exists" -ForegroundColor Gray
    }
    
    $libName = "Sample Document Library CLI"
    
    Write-Verbose "Checking if document library exists..."
    $libCheckResult = m365 spo list get --webUrl $script:SiteUrl --title $libName --output json 2>&1
    
    if ($LASTEXITCODE -ne 0) {
        Write-Host "Creating document library '$libName'..." -ForegroundColor Yellow
        
        if ($PSCmdlet.ShouldProcess($libName, 'Create document library')) {
            try {
                $libResult = m365 spo list add --webUrl $script:SiteUrl --title $libName --baseTemplate DocumentLibrary --output json
                if ($LASTEXITCODE -ne 0) {
                    throw "CLI error: $libResult"
                }
                $script:Summary.Library++
                Write-Host "✓ Document library created successfully" -ForegroundColor Green
            }
            catch {
                Write-Warning "Failed to create document library: $_"
                $script:Summary.Failures++
            }
        }
        else {
            $script:Summary.Library++
        }
    }
    else {
        Write-Host "✓ Document library '$libName' already exists" -ForegroundColor Gray
    }
    
    $folderName = "Sample Folder CLI"
    
    Write-Verbose "Checking if folder exists..."
    $folderCheckResult = m365 spo folder get --webUrl $script:SiteUrl --folderUrl "$libName/$folderName" --output json 2>&1
    
    if ($LASTEXITCODE -ne 0) {
        Write-Host "Creating folder '$folderName' in library '$libName'..." -ForegroundColor Yellow
        
        if ($PSCmdlet.ShouldProcess("$libName/$folderName", 'Create folder')) {
            try {
                $folderResult = m365 spo folder add --webUrl $script:SiteUrl --parentFolderUrl $libName --name $folderName --output json
                if ($LASTEXITCODE -ne 0) {
                    throw "CLI error: $folderResult"
                }
                $script:Summary.Folder++
                Write-Host "✓ Folder created successfully" -ForegroundColor Green
            }
            catch {
                Write-Warning "Failed to create folder: $_"
                $script:Summary.Failures++
            }
        }
        else {
            $script:Summary.Folder++
        }
    }
    else {
        Write-Host "✓ Folder '$folderName' already exists" -ForegroundColor Gray
    }
    
    Write-Host ""
    Write-Host "Configuring navigation..." -ForegroundColor Cyan
    Write-Host ""
    
    Write-Verbose "Adding navigation node for document library..."
    if ($PSCmdlet.ShouldProcess("QuickLaunch/$libName", 'Add navigation node')) {
        try {
            $navResult = m365 spo navigation node add --webUrl $script:SiteUrl --location QuickLaunch --title $libName --url "/$SiteRelativeUrl/$libName/Forms/AllItems.aspx" --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to add navigation node: $navResult"
                $script:Summary.Failures++
            }
            else {
                $script:Summary.Navigation++
                Write-Host "✓ Navigation node added for document library" -ForegroundColor Green
            }
        }
        catch {
            Write-Warning "Failed to add navigation node: $_"
            $script:Summary.Failures++
        }
    }
    else {
        $script:Summary.Navigation++
    }
}

end {
    Write-Host ""
    Write-Host "======================" -ForegroundColor Cyan
    Write-Host "Setup Complete" -ForegroundColor Cyan
    Write-Host "======================" -ForegroundColor Cyan
    Write-Host ""
    
    Write-Host "Summary:" -ForegroundColor White
    Write-Host "  Site Columns:      $($script:Summary.SiteColumn)" -ForegroundColor $(if ($script:Summary.SiteColumn -gt 0) { 'Green' } else { 'Gray' })
    Write-Host "  Content Types:     $($script:Summary.ContentType)" -ForegroundColor $(if ($script:Summary.ContentType -gt 0) { 'Green' } else { 'Gray' })
    Write-Host "  Lists:             $($script:Summary.List)" -ForegroundColor $(if ($script:Summary.List -gt 0) { 'Green' } else { 'Gray' })
    Write-Host "  Document Libraries: $($script:Summary.Library)" -ForegroundColor $(if ($script:Summary.Library -gt 0) { 'Green' } else { 'Gray' })
    Write-Host "  Folders:           $($script:Summary.Folder)" -ForegroundColor $(if ($script:Summary.Folder -gt 0) { 'Green' } else { 'Gray' })
    Write-Host "  Navigation Nodes:  $($script:Summary.Navigation)" -ForegroundColor $(if ($script:Summary.Navigation -gt 0) { 'Green' } else { 'Gray' })
    Write-Host "  Failures:          $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    
    Write-Host ""
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Gray
    
    Stop-Transcript
}

# Example 1: Create site with default settings
# .\\Setup-ExampleSite.ps1

# Example 2: Create site with custom relative URL and title
# .\\Setup-ExampleSite.ps1 -SiteRelativeUrl "sites/demo" -SiteTitle "Demo Site"

# Example 3: Test with WhatIf (see what would be created without making changes)
# .\\Setup-ExampleSite.ps1 -WhatIf

# Example 4: Run with verbose output for detailed progress
# .\\Setup-ExampleSite.ps1 -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***


## Contributors

| Author(s) |
|-----------|
| Valeras Narbutas |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-setup-example-site" aria-hidden="true" />

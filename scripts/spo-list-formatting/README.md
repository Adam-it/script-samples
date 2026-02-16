

# Export / Import list formatting

## Summary

This sample provides PowerShell scripts to export and import SharePoint list formatting customizations (column formatting, view formatting, and form customizers). Works with both **PnP PowerShell** and **CLI for Microsoft 365 v11.4.0+**.

SharePoint Online provides user interface for defining formatting:
- [Use view formatting to customize SharePoint](https://learn.microsoft.com/sharepoint/dev/declarative-customization/view-formatting), 
- [Show or hide columns in a list or library form](https://learn.microsoft.com/sharepoint/dev/declarative-customization/list-form-conditional-show-hide) and 
- [Configure the list form](https://learn.microsoft.com/sharepoint/dev/declarative-customization/list-form-configuration). 

However, the `Get-PnPSiteTemplate -Handlers Lists` command does not include these customizations. These scripts bridge this gap.

**Exported artifacts**:
- form customizers 
- list views
- list views formatting
- columns formatting 

to a set of JSON, CSV and XML files. Import scripts apply the exported customizations to an existing list.

![Example Screenshot](assets/example.png)

Use it with `Get-PnPSiteTemplate` and `Invoke-PnPSiteTemplate` for a full export / import capabilities.

# [PnP PowerShell](#tab/pnpps)

```powershell

function Get-ListFormatting {
    [CmdletBinding()]
    param (
        [Parameter()]
        [string]
        $listName,
        [string]
        $folderPath
    )

    $clientContext = Get-PnPContext
    $list = Get-PnPList $listName -Includes SchemaXml

    # Currently only Item content type is supported. The script may be easily adapted to enumerate through all the available content types
    $contentType = Get-PnPContentType -List $listName | Where-Object { $_.Name -eq "Item"}
    $clientContext.Load($list.Views)
    $clientContext.Load($contentType)
    $clientContext.Load($contentType.FieldLinks)
    $clientContext.ExecuteQuery()

    # 1. Get form customizer
    $contentType.ClientFormCustomFormatter | Out-File "$folderPath\ListFormatting.Form.$listName.json"
    # 2.Get fieldLinks settings
    $contentType.FieldLinks | Select-Object Name, Hidden, Id |  Export-Csv  "$folderPath\ListFormatting.ColumnOrder.$listName.csv" -NoTypeInformation

    # 3. Get list views 
    $listUrl=$list.RootFolder.ServerRelativeUrl+"/"
    $list.Views | Where-Object { $_.PersonalView -eq $false } | Select-Object Title, Id, @{name = "Url"; expression = { $_.ServerRelativeUrl.Replace($listUrl,"").Replace(".aspx","") } } | Export-Csv  "$folderPath\List.Views.$listName.csv" -NoTypeInformation
    $list.Views | ForEach-Object {
        $v = Get-PnPView -List $list -Identity $_.Id -Includes HtmlSchemaXml 
        $v.HtmlSchemaXml |  Out-File "$folderPath\List.View.$listName.$($_.Id).xml"
    }

    #get list views formatting
    $views = $list.Views | Where-Object { $null -ne $_.CustomFormatter } 
    if($null -ne $views){
        $views | Select-Object Title, Id | Export-Csv  "$folderPath\ListFormatting.Views.$listName.csv" -NoTypeInformation
        $views | ForEach-Object {
            $_.CustomFormatter | Out-File "$folderPath\ListFormatting.View.$listName.$($_.Id).json"
        }
    }

    #get columns formatting
    $columns= Get-PnPField -List $listName | Where-Object {$null -ne $_.CustomFormatter } 
    if($null -ne $columns){
        $columns | Select-Object InternalName | Export-Csv   "$folderPath\ListFormatting.Columns.$listName.csv" -NoTypeInformation
        $columns | ForEach-Object {
            $_.CustomFormatter | Out-File "$folderPath\ListFormatting.Column.$listName.$($_.InternalName).json"
        }
    }
}

function Set-ListFormatting {
    [CmdletBinding()]
    param (
        [Parameter()]
        [string]
        $listName,
        [string]
        $folderPath
    )
    if (Test-Path -Path $folderPath){

        Write-Host "LIST $listName "
        $clientContext = Get-PnPContext
        $list = Get-PnPList $listName

        #Get Content Type
        $contentType = Get-PnPContentType -List $listName | Where-Object { $_.Name -eq "Item" -or $_.Name -eq "Element" }
        $clientContext.Load($contentType)
        $clientContext.Load($contentType.FieldLinks)
        $clientContext.ExecuteQuery()

        # 1. Set form customizers
        if ($t=Test-Path -Path "$folderPath\ListFormatting.Form.$listName.json" -PathType leaf){
            Write-Host "Setting form customizers from $folderPath\ListFormatting.Form.$listName.json"
            $contentType.ClientFormCustomFormatter = (Get-Content -Raw -Path "$folderPath\ListFormatting.Form.$listName.json").ToString()
            Write-Host "...done"
        }
        # 2.a Set fieldLinks order
        if ($t=Test-Path -Path "$folderPath\ListFormatting.ColumnOrder.$listName.csv" -PathType leaf) {
            Write-Host "Setting fields order from $folderPath\ListFormatting.ColumnOrder.$listName.csv"

            $ColumnOrder = (Import-Csv "$folderPath\ListFormatting.ColumnOrder.$listName.csv").Name
            $contentType.FieldLinks.Reorder($ColumnOrder)
            Write-Host "...done"
        }
        # 2.b Set fieldLinks.Hidden 
        if ($t = Test-Path -Path "$folderPath\ListFormatting.ColumnOrder.$listName.csv" -PathType leaf) {
            Write-Host "Setting hidden fields from $folderPath\ListFormatting.ColumnOrder.$listName.csv"
            
            Import-Csv "$folderPath\ListFormatting.ColumnOrder.$listName.csv" | ForEach-Object{
                $contentType.FieldLinks.GetById($_.Id).Hidden = [System.Convert]::ToBoolean($_.Hidden)
            }
            Write-Host "...done"
        }
        $contentType.Update(0)
        $clientContext.ExecuteQuery()

        # 3. Set list views
        if ($t = Test-Path -Path  "$folderPath\List.Views.$listName.csv" -PathType leaf) {
            Write-Host "Setting views"
            
            #Get All List Views 
            $clientContext.Load($list.Views)
            $clientContext.ExecuteQuery()
            $listUrl=$list.RootFolder.ServerRelativeUrl+"/"
            $views = $list.Views | Select-Object Title, @{name = "Url"; expression = { $_.ServerRelativeUrl.Replace($listUrl, "").Replace(".aspx", "")}}

            Import-Csv "$folderPath\List.Views.$listName.csv" | ForEach-Object {

                $xml = [xml]( Get-Content "$folderPath\List.View.$listName.$($_.Id).xml" -Raw)
                $isDefault = [boolean]$xml.View.DefaultView
                $fields =  $xml.View.ViewFields.FieldRef.Name 
                $query = $xml.View.Query.InnerXml
                $aggregations = $xml.View.Aggregations.InnerXml
                $viewType2 = $xml.View.ViewType2

                $url = $_.Url
                $viewTitle = ($views | Where-Object { $_.Url -eq $url } ).Title

                if($null -ne $viewTitle){
                    Write-Host "Updating view $viewTitle to $($_.Title)"
                    $v = Set-PnPView -List $listName -Identity $viewTitle -Fields $fields -Values @{Title = $_.Title; ViewQuery = $query; ViewType2 = $viewType2 } -Aggregations $aggregations
                }
                else{
                    Write-Host "Creating view $($_.Title)"
                    #cannot set view url when creating the list. the following workaroud required
                    $v = Add-PnPView -List $listName -Title $_.Url  -SetAsDefault:$isDefault -Fields $fields -Query $query -Aggregations $aggregations
                    $v = Set-PnPView -List $listName -Identity $_.Url -Values @{Title = $_.Title ; ViewType2 = $viewType2} 
                }
            }
        }

        #Set list views formatting
        if ($t=Test-Path -Path "$folderPath\ListFormatting.Views.$listName.csv" -PathType leaf){
            Write-Host "Setting  list views formatting"

            #Get All List Views 
            $clientContext.Load($list.Views)
            $clientContext.ExecuteQuery()
            $views = $list.Views.Title

            #Get exported Views Info (Title & Id)
            Import-Csv "$folderPath\ListFormatting.Views.$listName.csv" | ForEach-Object{
                #If target list contains the view and the file exists
                if ($views.Contains($_.Title) ) {
                    # Update the List View Formatting Definition
                    $listViewFormattingJSON = Get-Content -Raw -Path "$folderPath\ListFormatting.View.$listName.$($_.Id).json";
                    $listViewColumnDefinition = Get-PnPView -List $listName -Identity $_.Title  
                    $listViewColumnDefinition | Set-PnPView -Values  @{CustomFormatter = $listViewFormattingJSON.ToString() }
                }
            }
        }
        #Set columns formatting
        if ($t=Test-Path -Path "$folderPath\ListFormatting.Columns.$listName.csv" -PathType leaf) {
            Write-Host "Setting columns formatting"

            #Get All List Columns
            $clientContext.Load($list.Fields)
            $clientContext.ExecuteQuery()
            $columns = $list.Fields.InternalName

            Import-Csv "$folderPath\ListFormatting.Columns.$listName.csv"  | ForEach-Object { #$columns.Contains($_.InternalName)
                if ($columns.Contains($_.InternalName)){
                    Write-Host "Setting formatter for $($_.InternalName)"
                    $ColumnFormattingJSON = Get-Content -Raw -Path "$folderPath\ListFormatting.Column.$listName.$($_.InternalName).json";
                    $listColumnDefinition = Get-PnPField -List $listName -Identity $_.InternalName
                    $listColumnDefinition | Set-PnPField -Values @{CustomFormatter = $ColumnFormattingJSON.ToString() }
                }
            }
        }
        
    }
}


### Example usage
$list1= "MyList1"
$folderPath= "C:\SPO\Templates"

Connect-PnPOnline -Credentials $PSCredentials -Url $siteUrl 

##Export
Get-PnPSiteTemplate  -Out "$folderPath/ListTemplates.xml" -ListsToExtract $list1 -Handlers Lists 
Get-ListFormatting -folderPath $folderPath -listName $list1

##Import
Invoke-PnPSiteTemplate -Path "$folderPath/ListTemplates.xml"
Set-ListFormatting -folderPath $folderPath -listName $list1


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(DefaultParameterSetName = 'Export', SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, ParameterSetName = 'Export')]
    [Parameter(Mandatory = $true, ParameterSetName = 'Import')]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory = $true, ParameterSetName = 'Export', HelpMessage = "List name or title")]
    [Parameter(Mandatory = $true, ParameterSetName = 'Import', HelpMessage = "List name or title")]
    [ValidateNotNullOrEmpty()]
    [string]$ListName,
    
    [Parameter(Mandatory = $true, ParameterSetName = 'Export', HelpMessage = "Folder path for export/import files")]
    [Parameter(Mandatory = $true, ParameterSetName = 'Import', HelpMessage = "Folder path for export/import files")]
    [ValidateScript({ Test-Path $_ -PathType Container })]
    [string]$FolderPath,
    
    [Parameter(Mandatory = $true, ParameterSetName = 'Export')]
    [switch]$Export,
    
    [Parameter(Mandatory = $true, ParameterSetName = 'Import')]
    [switch]$Import
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    
    Write-Verbose "Validating list '$ListName' exists at $SiteUrl..."
    $listJson = m365 spo list get --webUrl $SiteUrl --title $ListName --output json 2>$null
    if ($LASTEXITCODE -ne 0) {
        throw "List '$ListName' not found in site $SiteUrl"
    }
    $script:List = $listJson | ConvertFrom-Json
    Write-Verbose "List validated: $($script:List.Title) (ID: $($script:List.Id))"
    
    $script:Summary = @{
        FilesExported = 0
        FilesImported = 0
        Failures = 0
    }
    
    $logPath = Join-Path $FolderPath "ListFormatting_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    Start-Transcript -Path $logPath
    
    Write-Host "Starting $($PSCmdlet.ParameterSetName) operation for list '$ListName'..." -ForegroundColor Cyan
}

process {
    if ($Export) {
        Write-Host "`nExporting list formatting..." -ForegroundColor Yellow
        
        try {
            # 1. Export form customizer
            Write-Verbose "Getting content types for list '$ListName'..."
            $ctsJson = m365 spo contenttype list --webUrl $SiteUrl --listTitle $ListName --output json
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve content types"
            }
            
            $contentTypes = @($ctsJson | ConvertFrom-Json)
            $itemCT = $contentTypes | Where-Object { $_.Name -eq 'Item' -or $_.Name -eq 'Element' } | Select-Object -First 1
            
            if ($itemCT) {
                Write-Verbose "Found content type: $($itemCT.Name) (ID: $($itemCT.StringId))"
                
                if ($itemCT.ClientFormCustomFormatter) {
                    $formPath = Join-Path $FolderPath "ListFormatting.Form.$ListName.json"
                    Write-Host "  Exporting form customizer..." -ForegroundColor Green
                    $itemCT.ClientFormCustomFormatter | Out-File -FilePath $formPath -Encoding utf8
                    $script:Summary.FilesExported++
                } else {
                    Write-Verbose "No form customizer found for content type '$($itemCT.Name)'"
                }
                
                # 2. Export field links (order + hidden state)
                Write-Verbose "Getting field links for content type '$($itemCT.Name)'..."
                $fieldsJson = m365 spo contenttype field list --webUrl $SiteUrl --listTitle $ListName --contentTypeId $itemCT.StringId --output json
                if ($LASTEXITCODE -eq 0) {
                    $fields = @($fieldsJson | ConvertFrom-Json)
                    if ($fields.Count -gt 0) {
                        $fieldOrderPath = Join-Path $FolderPath "ListFormatting.ColumnOrder.$ListName.csv"
                        Write-Host "  Exporting field links ($($fields.Count) fields)..." -ForegroundColor Green
                        $fields | Select-Object Name, Hidden, Id | Export-Csv -Path $fieldOrderPath -NoTypeInformation
                        $script:Summary.FilesExported++
                    }
                }
            } else {
                Write-Warning "No 'Item' or 'Element' content type found in list"
            }
            
            # 3. Export views
            Write-Verbose "Getting views for list '$ListName'..."
            $viewsJson = m365 spo list view list --webUrl $SiteUrl --listTitle $ListName --output json --query "[?PersonalView == \`false\`]"
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve views"
            }
            
            $views = @($viewsJson | ConvertFrom-Json)
            Write-Host "  Found $($views.Count) non-personal views" -ForegroundColor Yellow
            
            if ($views.Count -gt 0) {
                $viewsPath = Join-Path $FolderPath "List.Views.$ListName.csv"
                $views | Select-Object Title, Id, @{Name='Url';Expression={$_.ServerRelativeUrl}} | Export-Csv -Path $viewsPath -NoTypeInformation
                $script:Summary.FilesExported++
                
                $viewIndex = 0
                foreach ($view in $views) {
                    $viewIndex++
                    Write-Progress -Activity "Exporting views" -Status "$($view.Title)" -PercentComplete (($viewIndex / $views.Count) * 100)
                    
                    Write-Verbose "Getting details for view '$($view.Title)' (ID: $($view.Id))..."
                    $viewDetailJson = m365 spo list view get --webUrl $SiteUrl --listTitle $ListName --id $view.Id --output json
                    if ($LASTEXITCODE -eq 0) {
                        $viewDetail = $viewDetailJson | ConvertFrom-Json
                        
                        # Export view schema
                        if ($viewDetail.HtmlSchemaXml) {
                            $viewSchemaPath = Join-Path $FolderPath "List.View.$ListName.$($view.Id).xml"
                            Write-Host "    Exporting view schema: $($view.Title)" -ForegroundColor Green
                            $viewDetail.HtmlSchemaXml | Out-File -FilePath $viewSchemaPath -Encoding utf8
                            $script:Summary.FilesExported++
                        }
                        
                        # Export view formatting (if exists)
                        if ($viewDetail.CustomFormatter) {
                            $viewFormattingPath = Join-Path $FolderPath "ListFormatting.View.$ListName.$($view.Id).json"
                            Write-Host "    Exporting view formatting: $($view.Title)" -ForegroundColor Green
                            $viewDetail.CustomFormatter | Out-File -FilePath $viewFormattingPath -Encoding utf8
                            $script:Summary.FilesExported++
                            
                            # Track formatted views for metadata CSV
                            if (-not $script:FormattedViews) {
                                $script:FormattedViews = [System.Collections.ArrayList]::new()
                            }
                            [void]$script:FormattedViews.Add($view)
                        }
                    } else {
                        Write-Warning "Failed to get details for view '$($view.Title)'"
                        $script:Summary.Failures++
                    }
                }
                
                # Export formatted views metadata CSV
                if ($script:FormattedViews -and $script:FormattedViews.Count -gt 0) {
                    $formattedViewsPath = Join-Path $FolderPath "ListFormatting.Views.$ListName.csv"
                    $script:FormattedViews | Select-Object Title, Id | Export-Csv -Path $formattedViewsPath -NoTypeInformation
                    $script:Summary.FilesExported++
                }
                
                Write-Progress -Activity "Exporting views" -Completed
            }
            
            # 4. Export column formatting
            Write-Verbose "Getting fields with custom formatting..."
            $columnsJson = m365 spo field list --webUrl $SiteUrl --listTitle $ListName --output json --query "[?CustomFormatter != null]"
            if ($LASTEXITCODE -eq 0) {
                $columns = @($columnsJson | ConvertFrom-Json)
                
                if ($columns.Count -gt 0) {
                    Write-Host "  Found $($columns.Count) columns with custom formatting" -ForegroundColor Yellow
                    
                    $columnsPath = Join-Path $FolderPath "ListFormatting.Columns.$ListName.csv"
                    $columns | Select-Object InternalName | Export-Csv -Path $columnsPath -NoTypeInformation
                    $script:Summary.FilesExported++
                    
                    foreach ($column in $columns) {
                        $columnFormattingPath = Join-Path $FolderPath "ListFormatting.Column.$ListName.$($column.InternalName).json"
                        Write-Host "    Exporting column formatting: $($column.InternalName)" -ForegroundColor Green
                        $column.CustomFormatter | Out-File -FilePath $columnFormattingPath -Encoding utf8
                        $script:Summary.FilesExported++
                    }
                } else {
                    Write-Verbose "No columns with custom formatting found"
                }
            }
            
            Write-Host "`nExport completed successfully!" -ForegroundColor Cyan
        }
        catch {
            $script:Summary.Failures++
            Write-Host "`nExport failed: $_" -ForegroundColor Red
            throw
        }
    }
    
    if ($Import) {
        Write-Host "`nImporting list formatting..." -ForegroundColor Yellow
        
        try {
            # 1. Import form customizer
            $formPath = Join-Path $FolderPath "ListFormatting.Form.$ListName.json"
            if (Test-Path $formPath) {
                Write-Verbose "Found form customizer file: $formPath"
                $formJSON = Get-Content -Path $formPath -Raw
                
                # Get Item content type ID first
                $ctsJson = m365 spo contenttype list --webUrl $SiteUrl --listTitle $ListName --output json
                if ($LASTEXITCODE -eq 0) {
                    $contentTypes = @($ctsJson | ConvertFrom-Json)
                    $itemCT = $contentTypes | Where-Object { $_.Name -eq 'Item' -or $_.Name -eq 'Element' } | Select-Object -First 1
                    
                    if ($itemCT -and $PSCmdlet.ShouldProcess("Form Customizer for $($itemCT.Name)", "Import")) {
                        Write-Host "  Importing form customizer..." -ForegroundColor Green
                        m365 spo contenttype set --webUrl $SiteUrl --listTitle $ListName --id $itemCT.StringId --ClientFormCustomFormatter $formJSON
                        if ($LASTEXITCODE -eq 0) {
                            $script:Summary.FilesImported++
                            Write-Verbose "Form customizer imported successfully"
                        } else {
                            Write-Warning "Failed to import form customizer"
                            $script:Summary.Failures++
                        }
                    }
                }
            } else {
                Write-Verbose "No form customizer file found: $formPath"
            }
            
            # 2. Import view formatting
            $viewsCSV = Join-Path $FolderPath "ListFormatting.Views.$ListName.csv"
            if (Test-Path $viewsCSV) {
                Write-Verbose "Found view formatting metadata: $viewsCSV"
                $formattedViews = Import-Csv $viewsCSV
                Write-Host "  Found $($formattedViews.Count) views with formatting to import" -ForegroundColor Yellow
                
                $viewIndex = 0
                foreach ($viewMeta in $formattedViews) {
                    $viewIndex++
                    Write-Progress -Activity "Importing view formatting" -Status "$($viewMeta.Title)" -PercentComplete (($viewIndex / $formattedViews.Count) * 100)
                    
                    $viewFormattingPath = Join-Path $FolderPath "ListFormatting.View.$ListName.$($viewMeta.Id).json"
                    if (Test-Path $viewFormattingPath) {
                        $viewJSON = Get-Content -Path $viewFormattingPath -Raw
                        
                        if ($PSCmdlet.ShouldProcess("View '$($viewMeta.Title)' formatting", "Import")) {
                            Write-Host "    Importing view formatting: $($viewMeta.Title)" -ForegroundColor Green
                            m365 spo list view set --webUrl $SiteUrl --listTitle $ListName --title $viewMeta.Title --CustomFormatter $viewJSON
                            if ($LASTEXITCODE -eq 0) {
                                $script:Summary.FilesImported++
                            } else {
                                Write-Warning "Failed to import formatting for view '$($viewMeta.Title)'"
                                $script:Summary.Failures++
                            }
                        }
                    } else {
                        Write-Warning "View formatting file not found: $viewFormattingPath"
                    }
                }
                
                Write-Progress -Activity "Importing view formatting" -Completed
            } else {
                Write-Verbose "No view formatting metadata file found: $viewsCSV"
            }
            
            # 3. Import column formatting
            $columnsCSV = Join-Path $FolderPath "ListFormatting.Columns.$ListName.csv"
            if (Test-Path $columnsCSV) {
                Write-Verbose "Found column formatting metadata: $columnsCSV"
                $formattedColumns = Import-Csv $columnsCSV
                Write-Host "  Found $($formattedColumns.Count) columns with formatting to import" -ForegroundColor Yellow
                
                foreach ($columnMeta in $formattedColumns) {
                    $columnFormattingPath = Join-Path $FolderPath "ListFormatting.Column.$ListName.$($columnMeta.InternalName).json"
                    if (Test-Path $columnFormattingPath) {
                        $columnJSON = Get-Content -Path $columnFormattingPath -Raw
                        
                        if ($PSCmdlet.ShouldProcess("Column '$($columnMeta.InternalName)' formatting", "Import")) {
                            Write-Host "    Importing column formatting: $($columnMeta.InternalName)" -ForegroundColor Green
                            m365 spo field set --webUrl $SiteUrl --listTitle $ListName --name $columnMeta.InternalName --CustomFormatter $columnJSON
                            if ($LASTEXITCODE -eq 0) {
                                $script:Summary.FilesImported++
                            } else {
                                Write-Warning "Failed to import formatting for column '$($columnMeta.InternalName)'"
                                $script:Summary.Failures++
                            }
                        }
                    } else {
                        Write-Warning "Column formatting file not found: $columnFormattingPath"
                    }
                }
            } else {
                Write-Verbose "No column formatting metadata file found: $columnsCSV"
            }
            
            Write-Host "`nImport completed successfully!" -ForegroundColor Cyan
        }
        catch {
            $script:Summary.Failures++
            Write-Host "`nImport failed: $_" -ForegroundColor Red
            throw
        }
    }
}

end {
    Write-Host "`n=== List Formatting Operation Summary ===" -ForegroundColor Cyan
    
    if ($Export) {
        Write-Host "Files Exported: " -NoNewline
        Write-Host $script:Summary.FilesExported -ForegroundColor Green
    } else {
        Write-Host "Files Imported: " -NoNewline
        Write-Host $script:Summary.FilesImported -ForegroundColor Green
    }
    
    Write-Host "Failures: " -NoNewline
    if ($script:Summary.Failures -gt 0) {
        Write-Host $script:Summary.Failures -ForegroundColor Red
    } else {
        Write-Host $script:Summary.Failures -ForegroundColor Green
    }
    
    Write-Host "Log File: " -NoNewline
    Write-Host $logPath -ForegroundColor Cyan
    Write-Host "========================================="n" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Example 1: Export list formatting to folder
# .\\Export-ImportListFormatting.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -ListName "Projects" -FolderPath "C:\\Temp\\Exports" -Export

# Example 2: Import list formatting with WhatIf preview
# .\\Export-ImportListFormatting.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Sales" -ListName "Projects" -FolderPath "C:\\Temp\\Exports" -Import -WhatIf

# Example 3: Import list formatting with verbose output
# .\\Export-ImportListFormatting.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Sales" -ListName "Projects" -FolderPath "C:\\Temp\\Exports" -Import -Verbose

# Example 4: Export with verbose output for troubleshooting
# .\\Export-ImportListFormatting.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -ListName "Custom List" -FolderPath "C:\\Exports" -Export -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]


## Contributors

| Author(s) |
|-----------|
| Kinga Kazala |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-list-formatting" aria-hidden="true" />

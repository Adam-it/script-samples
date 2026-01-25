

# Bulk import data from multiple files to multiple lists

## Summary
The script can help import test data in bulk into multiple lists in SharePoint Online using PnP PowerShell or CLI for Microsoft 365.

  ![Example Screenshot](assets/example.png)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "SharePoint site URL (e.g., https://contoso.sharepoint.com/sites/project)")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,

    [Parameter(Mandatory, HelpMessage = "Folder path containing CSV files (file names must match list titles)")]
    [ValidateScript({ Test-Path $_ -PathType Container })]
    [string]$CSVFolderPath,

    [Parameter(Mandatory, HelpMessage = "Array of list titles to import (e.g., @('Products', 'Feedback'))")]
    [string[]]$Lists
)

function Get-LookupID {
    param(
        [string]$SiteUrl,
        [string]$ListTitle,
        [string]$LookupFieldName,
        [string]$LookupValue
    )
    
    $cacheKey = "$ListTitle|$LookupFieldName|$LookupValue"
    if ($script:LookupCache.ContainsKey($cacheKey)) {
        return $script:LookupCache[$cacheKey]
    }
    
    try {
        $fieldCacheKey = "$ListTitle|$LookupFieldName"
        if (-not $script:FieldCache.ContainsKey($fieldCacheKey)) {
            $fieldJson = m365 spo field get --webUrl $SiteUrl --listId $script:ListIds[$ListTitle] --identity $LookupFieldName --output json
            if ($LASTEXITCODE -ne 0) { throw "Failed to get field metadata" }
            $script:FieldCache[$fieldCacheKey] = $fieldJson | ConvertFrom-Json
        }
        
        $field = $script:FieldCache[$fieldCacheKey]
        [xml]$schema = $field.SchemaXml
        $parentListID = $schema.Field.List
        $showField = $schema.Field.ShowField
        
        if (-not $showField) { $showField = "Title" }
        
        $parentCacheKey = "Parent|$parentListID"
        if (-not $script:ParentItemCache.ContainsKey($parentCacheKey)) {
            $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listId $parentListID --fields "Id,$showField" --output json
            if ($LASTEXITCODE -ne 0) { throw "Failed to query parent list" }
            $script:ParentItemCache[$parentCacheKey] = @($itemsJson | ConvertFrom-Json)
        }
        
        $items = $script:ParentItemCache[$parentCacheKey]
        $match = $items | Where-Object { $_.$showField -eq $LookupValue } | Select-Object -First 1
        
        $result = if ($match) { $match.Id } else { $null }
        $script:LookupCache[$cacheKey] = $result
        return $result
    }
    catch {
        Write-Warning "Failed to resolve lookup value '$LookupValue' for field '$LookupFieldName': $_"
        return $null
    }
}

begin {
    $transcriptPath = Join-Path $CSVFolderPath "BulkImport_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) { 
        throw "Failed to login to Microsoft 365. Please run 'm365 login' to authenticate." 
    }
    
    Write-Host "Retrieving list IDs..." -ForegroundColor Cyan
    $script:ListIds = @{}
    foreach ($list in $Lists) {
        try {
            $listJson = m365 spo list get --webUrl $SiteUrl --title $list --output json
            if ($LASTEXITCODE -ne 0) { throw "Failed to get list '$list'" }
            $listData = $listJson | ConvertFrom-Json
            $script:ListIds[$list] = $listData.Id
            Write-Verbose "List '$list' ID: $($listData.Id)"
        }
        catch {
            throw "Failed to retrieve list ID for '$list': $_"
        }
    }
    
    $script:Summary = @{}
    $script:LookupCache = @{}
    $script:FieldCache = @{}
    $script:ParentItemCache = @{}
    foreach ($list in $Lists) {
        $script:Summary[$list] = @{
            CSVRows = 0
            ItemsImported = 0
            Failures = 0
        }
    }
}

process {
    foreach ($listName in $Lists) {
        Write-Host "`nProcessing list: $listName" -ForegroundColor Cyan
        
        $csvFilePath = Join-Path $CSVFolderPath "$listName.csv"
        
        if (-not (Test-Path $csvFilePath)) {
            Write-Warning "CSV file not found: $csvFilePath. Skipping list '$listName'."
            continue
        }
        
        try {
            $csvData = Import-Csv $csvFilePath
            $script:Summary[$listName].CSVRows = $csvData.Count
            
            Write-Host "Found $($csvData.Count) rows in CSV file" -ForegroundColor Gray
            
            Write-Host "Retrieving list fields..." -ForegroundColor Gray
            $fieldsJson = m365 spo field list --webUrl $SiteUrl --listId $script:ListIds[$listName] --query "[?!(Hidden) && !(ReadOnlyField) && InternalName != 'ContentType' && InternalName != 'Attachments']" --output json
            if ($LASTEXITCODE -ne 0) { throw "Failed to retrieve fields for list '$listName'" }
            
            $listFields = @($fieldsJson | ConvertFrom-Json)
            
            Write-Host "Found $($listFields.Count) editable fields" -ForegroundColor Gray
            
            foreach ($row in $csvData) {
                $csvFields = $row.PSObject.Properties.Name
                
                $args = @(
                    'spo', 'listitem', 'add',
                    '--webUrl', $SiteUrl,
                    '--listId', $script:ListIds[$listName],
                    '--output', 'json'
                )
                
                foreach ($csvField in $csvFields) {
                    $mappedField = $listFields | Where-Object { $_.InternalName -eq $csvField }
                    
                    if ($null -eq $mappedField) { continue }
                    
                    $fieldName = $mappedField.InternalName
                    $fieldValue = $row.$csvField
                    
                    if ([string]::IsNullOrWhiteSpace($fieldValue)) { continue }
                    
                    $fieldType = $mappedField.TypeAsString
                    
                    switch ($fieldType) {
                        'User' {
                            $userEmail = $fieldValue.Trim()
                            $args += "--$fieldName"
                            $args += "[{'Key':'i:0#.f|membership|$userEmail'}]"
                        }
                        'UserMulti' {
                            $userEmails = $fieldValue.Split(',') | ForEach-Object { $_.Trim() }
                            $userArray = $userEmails | ForEach-Object { "{'Key':'i:0#.f|membership|$_'}" }
                            $args += "--$fieldName"
                            $args += "[$($userArray -join ',')]"
                        }
                        'Lookup' {
                            $lookupID = Get-LookupID -SiteUrl $SiteUrl -ListTitle $listName -LookupFieldName $fieldName -LookupValue $fieldValue
                            if ($lookupID) {
                                $args += "--$fieldName"
                                $args += $lookupID
                            }
                        }
                        'LookupMulti' {
                            $lookupValues = $fieldValue.Split(',') | ForEach-Object { $_.Trim() }
                            $lookupIDs = @()
                            foreach ($val in $lookupValues) {
                                $id = Get-LookupID -SiteUrl $SiteUrl -ListTitle $listName -LookupFieldName $fieldName -LookupValue $val
                                if ($id) { $lookupIDs += $id }
                            }
                            if ($lookupIDs.Count -gt 0) {
                                $args += "--$fieldName"
                                $args += ($lookupIDs -join ',')
                            }
                        }
                        'DateTime' {
                            try {
                                try {
                                    $datetime = [datetime]::ParseExact($fieldValue, 'dd/MM/yyyy HH:mm:ss', $null)
                                    $formattedDate = $datetime.ToString('yyyy-MM-dd HH:mm:ss')
                                }
                                catch {
                                    $datetime = [datetime]::ParseExact($fieldValue, 'dd/MM/yyyy', $null)
                                    $formattedDate = $datetime.ToString('yyyy-MM-dd')
                                }
                                $args += "--$fieldName"
                                $args += $formattedDate
                            }
                            catch {
                                Write-Warning "Invalid date format for field '$fieldName': $fieldValue"
                            }
                        }
                        { $_ -match 'Choice' -and $_ -match 'Multi' } {
                            $args += "--$fieldName"
                            $args += ($fieldValue -replace ',', ';#')
                        }
                        'TaxonomyFieldType' {
                            $args += "--$fieldName"
                            $args += $fieldValue
                        }
                        'TaxonomyFieldTypeMulti' {
                            $args += "--$fieldName"
                            $args += $fieldValue
                        }
                        default {
                            $args += "--$fieldName"
                            $args += $fieldValue
                        }
                    }
                }
                
                $targetDescription = "List: $listName"
                if ($PSCmdlet.ShouldProcess($targetDescription, 'Add list item')) {
                    try {
                        $itemJson = m365 @args
                        if ($LASTEXITCODE -ne 0) { throw "CLI returned error code $LASTEXITCODE" }
                        
                        $item = $itemJson | ConvertFrom-Json
                        Write-Verbose "Created item ID: $($item.Id)"
                        $script:Summary[$listName].ItemsImported++
                    }
                    catch {
                        Write-Warning "Failed to import row: $_"
                        $script:Summary[$listName].Failures++
                        continue
                    }
                }
                else {
                    $script:Summary[$listName].ItemsImported++
                }
            }
        }
        catch {
            Write-Warning "Error processing list '$listName': $_"
            $script:Summary[$listName].Failures = $script:Summary[$listName].CSVRows
        }
    }
}

end {
    Stop-Transcript
    
    Write-Host "`n" -NoNewline
    Write-Host "=== Import Summary ===" -ForegroundColor Cyan
    
    foreach ($listName in $Lists) {
        $stats = $script:Summary[$listName]
        $color = if ($stats.Failures -eq 0) { 'Green' } 
                 elseif ($stats.Failures -eq $stats.CSVRows) { 'Red' } 
                 else { 'Yellow' }
        
        Write-Host "`n$listName:" -ForegroundColor White
        Write-Host "  CSV Rows: $($stats.CSVRows)" -ForegroundColor Gray
        Write-Host "  Items Imported: $($stats.ItemsImported)" -ForegroundColor $color
        Write-Host "  Failures: $($stats.Failures)" -ForegroundColor $(if ($stats.Failures -gt 0) { 'Red' } else { 'Gray' })
    }
    
    Write-Host "`nTranscript saved to: $transcriptPath" -ForegroundColor Cyan
    
    if ($WhatIfPreference) {
        Write-Host "`nWhatIf mode - No items were actually imported" -ForegroundColor Yellow
    }
}

# Example 1: Import data to a single list
# ./Invoke-SPOBulkImport.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -CSVFolderPath "C:\Data" -Lists @("Products")

# Example 2: Import data to multiple lists
# ./Invoke-SPOBulkImport.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -CSVFolderPath "C:\Data" -Lists @("Product Types", "Managers", "Products", "Product Feedback")

# Example 3: Test import with WhatIf
# ./Invoke-SPOBulkImport.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -CSVFolderPath "C:\Data" -Lists @("Products") -WhatIf

# Example 4: Import with verbose output
# ./Invoke-SPOBulkImport.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -CSVFolderPath "C:\Data" -Lists @("Products") -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell
#The script is to import sample data into a test environment for one or multiple lists
$config = @{ lists = @("Product Types","Managers", "Products","Product Feedback" ) }


#Config Variables
$SiteURL = "https://tenant.sharepoint.com/teams/test"
#the file names need to match the list names provided in the lists array.
$path = "E:\Sample_Data\"


#Function to get Lookup ID from Lookup Value
Function Get-LookupID($listName, $LookupFieldName, $LookupValue)
{
    #Get Parent Lookup List and Field from Child Lookup Field's Schema XML
    $LookupField =  Get-PnPField -List $listName -Identity $LookupFieldName
    [Xml]$Schema = $LookupField.SchemaXml
    $ParentListID = $Schema.Field.Attributes["List"].'#text'
    $ParentField  = $Schema.field.Attributes["ShowField"].'#text'
    $ParentLookupItem  = Get-PnPListItem -List $ParentListID -Fields $ParentField | Where {$_[$ParentField] -eq $LookupValue} | Select -First 1
    If($ParentLookupItem -ne $Null)  { Return $ParentLookupItem["ID"] }  Else  { Return $Null }
}

Try {
    #Connect to the Site
    Connect-PnPOnline -URL $SiteURL -Interactive


    foreach($listName in $config.lists){
      $CSVFilePath = "{0}{1}{2}{3}" -f  $path,"\",$listName ,".csv"
      #Get the data from CSV file
      $CSVData = Import-CSV $CSVFilePath

      #Get the List to Add Items
      $List = Get-PnPList -Identity $listName

       #Get fields to Update from the List - Skip Read only, hidden fields, content type and attachments
       $ListFields = Get-PnPField -List $listName | Where { (-Not ($_.ReadOnlyField)) -and (-Not ($_.Hidden)) -and ($_.InternalName -ne  "ContentType") -and ($_.InternalName -ne  "Attachments") }

       #Loop through each Row in the CSV file and update the matching list item ID
       ForEach($Row in $CSVData){
         #Frame the List Item to update
         $ItemValue = @{}           
         $CSVFields = $Row | Get-Member -MemberType NoteProperty | Select -ExpandProperty Name
         #Map each field from CSV to target list
         Foreach($CSVField in $CSVFields){
            $MappedField = $ListFields | Where {$_.InternalName -eq $CSVField}
            If($MappedField -ne $Null){
               $FieldName = $MappedField.InternalName
                #Check if the Field value is not Null
                If($Row.$CSVField -ne $Null){
                    #Handle Special Fields
                    $FieldType  = $MappedField.TypeAsString
                    If($FieldType -eq "User" -or $FieldType -eq "UserMulti"){ #People Picker Field
                        $PeoplePickerValues = $Row.$FieldName.Split(",")
                        $ItemValue.add($FieldName,$PeoplePickerValues)
                    }
                    ElseIf($FieldType -eq "Lookup" -or $FieldType -eq "LookupMulti"){ #Lookup Field
                        $LookupIDs = $Row.$FieldName.Split(",") | ForEach-Object { Get-LookupID -ListName $listName -LookupFieldName $FieldName -LookupValue $_ }               
                        $ItemValue.Add($FieldName,$LookupIDs)
                    }
                    ElseIf($FieldType -eq "DateTime"){
                       if($Row.$FieldName -ne ""){
                          $datetime = [datetime]::ParseExact( $Row.$FieldName, 'dd/MM/yyyy', $null)
                         $ItemValue.Add($FieldName,$datetime)
                      }
                    }
                    Else{
                        #Get Source Field Value and add to Hashtable
                        $ItemValue.Add($FieldName,$Row.$FieldName)
                    }
                }
            }
        }
        Write-host "Adding List item with values:"
        $ItemValue | Format-Table
        #Add New List Item
        Add-PnPListItem -List $listName -Values $ItemValue | Out-Null
    }
  }
}
Catch {
    write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Contributors

| Author(s) |
|-----------|
| Reshmee Auckloo |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-bulk-import-data" aria-hidden="true" />

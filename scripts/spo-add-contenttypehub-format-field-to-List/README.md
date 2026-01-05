

# Add Content Type Hub with calendar format field to List

## Summary

 This script will create Content Type Hub with custom calendar List field formatting and include in destination site and associated custom List. 
 - Retrieve custom calendar Formatting json from [https://github.com/pnp/List-Formatting/](https://github.com/pnp/List-Formatting/).
 - Creates custom content type in Hub site.
 - Adds new field "CalendarDemo" to new content type Hub.
 - Publish custom Content Type and add to destination site.
 - Create Custom Lists with enable content type.
 - Remove default Item Content and include custom Content Type from Hub.
 - Add fields "Title,CalendarDemo" in default View.

More about List Formatting github repository.
 [https://github.com/pnp/List-Formatting/](https://github.com/pnp/List-Formatting/)

![Example Screenshot](assets/ContentTypeHubFormatField.gif)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory=$false, HelpMessage="JSON format URL for custom calendar field")]
    [string]$FieldFormatUrl = "https://raw.githubusercontent.com/pnp/List-Formatting/25e27c252be744fabe6ea312ccc526b4d676fbae/column-samples/generic-neumorphism/generic-neumorphism-calendar.json",
    
    [Parameter(Mandatory=$false, HelpMessage="Custom Content Type Name")]
    [string]$ContentTypeName = "Custom calendar Format",
    
    [Parameter(Mandatory=$false, HelpMessage="List Name to create")]
    [string]$ListName = "CLI List calendar Format",
    
    [Parameter(Mandatory=$true, HelpMessage="Destination Site URL")]
    [ValidatePattern('^https://.*')]
    [string]$DestinationSiteUrl
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    Start-Transcript -Path "ContentTypeHub-$timestamp-Transcript.log"
    
    Write-Host "Logging in to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to log in to Microsoft 365"
    }
    
    $script:Summary = @{
        HubUrlRetrieved = 0
        JsonDownloaded = 0
        ContentTypeCreated = 0
        FieldCreated = 0
        FieldFormatterApplied = 0
        FieldAddedToCT = 0
        ContentTypeSynced = 0
        ListCreated = 0
        DefaultCTRemoved = 0
        CustomCTAdded = 0
        ViewFieldsAdded = 0
        Failures = 0
    }
}

process {
    try {
        Write-Host "Step 1: Retrieving Content Type Hub URL..." -ForegroundColor Yellow
        $hubResult = m365 spo contenttypehub get --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to get Content Type Hub URL. Error: $hubResult"
            $script:Summary.Failures++
            return
        }
        $hubUrl = ($hubResult | ConvertFrom-Json).ContentTypePublishingHub
        Write-Host "  Hub URL: $hubUrl" -ForegroundColor Green
        $script:Summary.HubUrlRetrieved = 1
        
        Write-Host "Step 2: Downloading calendar field formatter JSON..." -ForegroundColor Yellow
        try {
            $formattingCalendarDemo = Invoke-WebRequest -Uri $FieldFormatUrl -ErrorAction Stop
            Write-Host "  JSON downloaded successfully" -ForegroundColor Green
            $script:Summary.JsonDownloaded = 1
        }
        catch {
            Write-Warning "Failed to download JSON from $FieldFormatUrl. Error: $_"
            $script:Summary.Failures++
            return
        }
        
        Write-Host "Step 3: Getting parent 'Item' Content Type from Hub..." -ForegroundColor Yellow
        $parentCTResult = m365 spo contenttype get --webUrl $hubUrl --name "Item" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to get parent 'Item' Content Type. Error: $parentCTResult"
            $script:Summary.Failures++
            return
        }
        $parentCT = $parentCTResult | ConvertFrom-Json
        $parentCTId = $parentCT.StringId
        Write-Host "  Parent CT ID: $parentCTId" -ForegroundColor Green
        
        Write-Host "Step 4: Generating new Content Type ID..." -ForegroundColor Yellow
        $newCTGuid = (New-Guid).Guid.Replace("-", "").ToUpper().Substring(0, 32)
        $newCTId = "$parentCTId$newCTGuid"
        Write-Host "  New CT ID: $newCTId" -ForegroundColor Green
        
        Write-Host "Step 5: Creating custom Content Type '$ContentTypeName' in Hub..." -ForegroundColor Yellow
        $addCTResult = m365 spo contenttype add --webUrl $hubUrl --id $newCTId --name $ContentTypeName --group $ContentTypeName --description "Content Type for $ContentTypeName" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to create Content Type. Error: $addCTResult"
            $script:Summary.Failures++
            return
        }
        Write-Host "  Content Type created successfully" -ForegroundColor Green
        $script:Summary.ContentTypeCreated = 1
        
        Write-Host "Step 6: Creating 'CalendarDemo' field in Hub..." -ForegroundColor Yellow
        $fieldGuid = (New-Guid).Guid.ToUpper()
        $fieldXml = "<Field Type='DateTime' DisplayName='CalendarDemo' Required='FALSE' EnforceUniqueValues='FALSE' Indexed='FALSE' Format='DateTime' Group='Custom Columns' FriendlyDisplayFormat='Disabled' ID='{$fieldGuid}' StaticName='CalendarDemo' Name='CalendarDemo'><Default>[today]</Default></Field>"
        $addFieldResult = m365 spo field add --webUrl $hubUrl --xml $fieldXml --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to create field. Error: $addFieldResult"
            $script:Summary.Failures++
            return
        }
        Write-Host "  Field created with ID: $fieldGuid" -ForegroundColor Green
        $script:Summary.FieldCreated = 1
        
        Write-Host "Step 7: Applying custom formatter to 'CalendarDemo' field..." -ForegroundColor Yellow
        $customFormatterJson = $formattingCalendarDemo.Content.Replace('"', '\"')
        m365 spo field set --webUrl $hubUrl --id $fieldGuid --CustomFormatter $customFormatterJson 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to apply custom formatter to field"
            $script:Summary.Failures++
        }
        else {
            Write-Host "  Custom formatter applied successfully" -ForegroundColor Green
            $script:Summary.FieldFormatterApplied = 1
        }
        
        Write-Host "Step 8: Adding 'CalendarDemo' field to Content Type..." -ForegroundColor Yellow
        m365 spo contenttype field set --webUrl $hubUrl --contentTypeId $newCTId --id $fieldGuid 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add field to Content Type"
            $script:Summary.Failures++
        }
        else {
            Write-Host "  Field added to Content Type successfully" -ForegroundColor Green
            $script:Summary.FieldAddedToCT = 1
        }
        
        Write-Host "Step 9: Syncing Content Type to destination site '$DestinationSiteUrl'..." -ForegroundColor Yellow
        m365 spo contenttype sync --webUrl $DestinationSiteUrl --id $newCTId --output json 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to sync Content Type to destination site"
            $script:Summary.Failures++
            return
        }
        Write-Host "  Content Type synced successfully (may take a few seconds to propagate)" -ForegroundColor Green
        $script:Summary.ContentTypeSynced = 1
        
        Write-Host "Step 10: Creating list '$ListName' in destination site..." -ForegroundColor Yellow
        $addListResult = m365 spo list add --webUrl $DestinationSiteUrl --title $ListName --baseTemplate GenericList --contentTypesEnabled true --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to create list. Error: $addListResult"
            $script:Summary.Failures++
            return
        }
        Write-Host "  List created successfully" -ForegroundColor Green
        $script:Summary.ListCreated = 1
        
        Write-Host "Step 11: Removing default 'Item' Content Type from list..." -ForegroundColor Yellow
        m365 spo list contenttype remove --webUrl $DestinationSiteUrl --listTitle $ListName --id $parentCTId --force 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to remove default 'Item' Content Type"
            $script:Summary.Failures++
        }
        else {
            Write-Host "  Default 'Item' Content Type removed successfully" -ForegroundColor Green
            $script:Summary.DefaultCTRemoved = 1
        }
        
        Write-Host "Step 12: Adding custom Content Type '$ContentTypeName' to list..." -ForegroundColor Yellow
        m365 spo list contenttype add --webUrl $DestinationSiteUrl --listTitle $ListName --id $newCTId --output json 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add custom Content Type to list"
            $script:Summary.Failures++
        }
        else {
            Write-Host "  Custom Content Type added to list successfully" -ForegroundColor Green
            $script:Summary.CustomCTAdded = 1
        }
        
        Write-Host "Step 13: Getting default view of list..." -ForegroundColor Yellow
        $viewListResult = m365 spo list view list --webUrl $DestinationSiteUrl --listTitle $ListName --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to get list views. Error: $viewListResult"
            $script:Summary.Failures++
            return
        }
        $views = @($viewListResult | ConvertFrom-Json)
        $defaultView = $views | Where-Object { $_.DefaultView -eq $true }
        if ($null -eq $defaultView) {
            Write-Warning "No default view found for list"
            $script:Summary.Failures++
            return
        }
        $defaultViewTitle = $defaultView.Title
        Write-Host "  Default view: $defaultViewTitle" -ForegroundColor Green
        
        Write-Host "Step 14: Adding 'Title' field to default view..." -ForegroundColor Yellow
        m365 spo list view field add --webUrl $DestinationSiteUrl --listTitle $ListName --viewTitle $defaultViewTitle --title "Title" 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add 'Title' field to view (may already exist)"
        }
        else {
            Write-Host "  'Title' field added to view" -ForegroundColor Green
        }
        
        Write-Host "Step 15: Adding 'CalendarDemo' field to default view..." -ForegroundColor Yellow
        m365 spo list view field add --webUrl $DestinationSiteUrl --listTitle $ListName --viewTitle $defaultViewTitle --title "CalendarDemo" 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add 'CalendarDemo' field to view"
            $script:Summary.Failures++
        }
        else {
            Write-Host "  'CalendarDemo' field added to view" -ForegroundColor Green
            $script:Summary.ViewFieldsAdded = 1
        }
        
        Write-Host "`nContent Type Hub setup completed!" -ForegroundColor Cyan
    }
    catch {
        Write-Warning "Unexpected error: $_"
        $script:Summary.Failures++
    }
}

end {
    Write-Host "`n=== Summary ===" -ForegroundColor Cyan
    Write-Host "Hub URL Retrieved       : $($script:Summary.HubUrlRetrieved)" -ForegroundColor $(if ($script:Summary.HubUrlRetrieved -eq 1) { 'Green' } else { 'Red' })
    Write-Host "JSON Downloaded         : $($script:Summary.JsonDownloaded)" -ForegroundColor $(if ($script:Summary.JsonDownloaded -eq 1) { 'Green' } else { 'Red' })
    Write-Host "Content Type Created    : $($script:Summary.ContentTypeCreated)" -ForegroundColor $(if ($script:Summary.ContentTypeCreated -eq 1) { 'Green' } else { 'Red' })
    Write-Host "Field Created           : $($script:Summary.FieldCreated)" -ForegroundColor $(if ($script:Summary.FieldCreated -eq 1) { 'Green' } else { 'Red' })
    Write-Host "Field Formatter Applied : $($script:Summary.FieldFormatterApplied)" -ForegroundColor $(if ($script:Summary.FieldFormatterApplied -eq 1) { 'Green' } else { 'Red' })
    Write-Host "Field Added to CT       : $($script:Summary.FieldAddedToCT)" -ForegroundColor $(if ($script:Summary.FieldAddedToCT -eq 1) { 'Green' } else { 'Red' })
    Write-Host "Content Type Synced     : $($script:Summary.ContentTypeSynced)" -ForegroundColor $(if ($script:Summary.ContentTypeSynced -eq 1) { 'Green' } else { 'Red' })
    Write-Host "List Created            : $($script:Summary.ListCreated)" -ForegroundColor $(if ($script:Summary.ListCreated -eq 1) { 'Green' } else { 'Red' })
    Write-Host "Default CT Removed      : $($script:Summary.DefaultCTRemoved)" -ForegroundColor $(if ($script:Summary.DefaultCTRemoved -eq 1) { 'Green' } else { 'Red' })
    Write-Host "Custom CT Added         : $($script:Summary.CustomCTAdded)" -ForegroundColor $(if ($script:Summary.CustomCTAdded -eq 1) { 'Green' } else { 'Red' })
    Write-Host "View Fields Added       : $($script:Summary.ViewFieldsAdded)" -ForegroundColor $(if ($script:Summary.ViewFieldsAdded -eq 1) { 'Green' } else { 'Red' })
    Write-Host "Failures                : $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -eq 0) { 'Green' } else { 'Red' })
    
    Stop-Transcript
}
```


# Example 1: Create content type with default settings
# .\Create-ContentTypeHub.ps1 -DestinationSiteUrl "https://contoso.sharepoint.com/sites/project"

# Example 2: Create content type with custom name and list name
# .\Create-ContentTypeHub.ps1 -DestinationSiteUrl "https://contoso.sharepoint.com/sites/project" -ContentTypeName "My Calendar CT" -ListName "My Calendar List"

# Example 3: Use custom field formatter JSON from different URL with verbose output
# .\Create-ContentTypeHub.ps1 -DestinationSiteUrl "https://contoso.sharepoint.com/sites/project" -FieldFormatUrl "https://example.com/custom-calendar.json" -Verbose

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell
[CmdletBinding()]
param(
  [Parameter( Mandatory=$false,HelpMessage="json format custom calendar field.")]
  [string]$FieldFormatUrl="https://raw.githubusercontent.com/pnp/List-Formatting/25e27c252be744fabe6ea312ccc526b4d676fbae/column-samples/generic-neumorphism/generic-neumorphism-calendar.json",
  [Parameter(Mandatory=$false,HelpMessage="Custom Content Type Name")]
  [String]$ContentTypeName="Custom calendar Format",
  [Parameter(Mandatory=$false,HelpMessage="List Name to create")]
  [String]$ListName="PnP List calendar Format",
  [Parameter(Mandatory=$true,HelpMessage="Destination Site Url")]
  [String]$DestinationSiteUrl="https://contoso.sharepoint.com"
)

    Begin
    {
        #Connect to destination SharePoint Site
        Connect-PnPOnline -Url $DestinationSiteUrl -Interactive
        
        #Get Content Type Hub Site associated to destination site   
        $HubUrl = Get-PnPContentTypePublishingHubUrl

        #Connect to destination SharePoint Site Hub        
        Connect-PnPOnline -Url $HubUrl -Interactive
    }
    Process
    {
        try
        {
            #https://github.com/pnp/List-Formatting/pull/559/files
            #Get Calendar json Formatting from Github List Repository
            $formattingCalendarDemo = Invoke-WebRequest -Uri $FieldFormatUrl

            #Content Type Name
            #Content Type Based on CT "Item"
            #Create new Content Type 
            $ContentTypeGroupName = $ContentTypeName
            $ParentContentType = Get-PnPContentType | Where-Object { $_.Name -eq "Item" }
            $CustomContentType = Add-PnPContentType -Name $ContentTypeGroupName -Description $ContentTypeGroupName -Group $ContentTypeGroupName -ParentContentType $ParentContentType

            #Create field "CalendarDemo"
            #Add list formatting "formattingCalendarDemo"
            $Customfield= Add-PnPField -Type DateTime -InternalName "CalendarDemo" -DisplayName "CalendarDemo" -Group "Custom Columns"
            $Field | Set-PnPField -Values @{CustomFormatter = $formattingCalendarDemo.Content; DefaultValue="[today]" } -Identity $Customfield.Id
            Add-PnPFieldToContentType -Field $Customfield.Id -ContentType $CustomContentType.Name

            #Publish Content Type in Hub and destination site
            Publish-PnPContentType -ContentType $CustomContentType.Id 
            Add-PnPContentTypesFromContentTypeHub -ContentTypes $CustomContentType.Id -Site $DestinationSiteUrl

            #Connect to DestinationSiteUrl
            Connect-PnPOnline -Url $DestinationSiteUrl -Interactive
            
            #Create List with EnableContentTypes
            New-PnPList -Title $ListName -Template GenericList -Url "lists/$($ListName.Replace(" ","""))" -EnableContentTypes  | Out-Null
            
            #Remove default Item Content Type
            #Add Content Type to List
            Remove-PnPContentTypeFromList -List $ListName -ContentType "Item"
            Add-PnPContentTypeToList -List $ListName -ContentType $ContentTypeName
        
            #Add fields to default View
            Set-PnPView -List $ListName -Identity (Get-PnPView -List $ListName).Id -Fields "Title","CalendarDemo" | Out-Null
        }
        catch{
            Write-Output "Something threw an exception or used Write-Error"
            Write-Output $_
        }
        finally{
            # Disconnect the context  
            Disconnect-PnPOnline  
        }
    }
    End
    {
        # Disconnect the context  
        Disconnect-PnPOnline  
        Write-Host "END" -BackgroundColor Red
    }

```

## Results running the script 
When accessing to Content Type Hub the following content should be available
![Content Type Hub Screenshot](assets/ContentTypeHub.PNG)

When accessing to created custom List the new Content Type Hub should be available.
![Content Type List Screenshot](assets/ContentTypeHubList.PNG)


[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Contributors

| Author(s) |
|-----------|
| [André Lage](https://github.com/aaclage) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-add-contenttypehub-format-field-to-List" aria-hidden="true" />

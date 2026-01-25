

# Adding a new modern calendar view to a SharePoint list using PnP PowerShell
# Adding a new modern calendar view to a SharePoint list

## Summary

Recently we have finally been able to add a modern calendar view to a list in SharePoint Online but only through the UI. Before this a calendar view was only available in SharePoint classic mode.

![Example Screenshot](assets/example.png)

This script allows you to add a new modern calendar view to an existing SharePoint list.

# [PnP PowerShell](#tab/pnpps)

This PnP PowerShell script uses the SharePoint REST API to add the view using the PnP cmdlet **Invoke-PnPSPRestMethod** as currently modern calendar view is not available using just **Add-PnPView**.

Key points to note regarding the JSON body:

* **RowLimit** is set to zero – this is to ensure all items for the current month/week/day are fetched correctly.
* **StartDate** (internal field name) is mapped to 0th entry in ViewFields
* **EndDate** (internal field name) is mapped to 1st entry in ViewFields
* **ViewData** has 5 FieldRef entries – 1 for month view and 2 each for week and day view. The fields are used as 'Title' for respective visualizations. If this is missing, you will see the popup to 'fix' calendar view.
* **CalendarViewStyles** has 3 CalendarViewStyle entry – will be used in future. Even if this is missing, View creation will succeed.
* **ViewType2** is MODERNCALENDAR
* **ViewTypeKind** is 1 – which maps to HTML.
* **Query** can be set if required.

```powershell
###### Declare and Initialize Variables ######  

$url = 'https://<tenant>.sharepoint.com/sites/sitename'
$listname = "Calendar" #Change to the SharePoint list name to be used
$newViewTitle = "Modern Calendar View" #Change if you require a different View name


## Connect to SharePoint Online site  
Connect-PnPOnline -Url $url -Interactive

$viewCreationJson = @"
{
    "parameters": {
        "__metadata": {
            "type": "SP.ViewCreationInformation"
        },
        "Title": "$newViewTitle",
        "ViewFields": {
            "__metadata": {
                "type": "Collection(Edm.String)"
            },
            "results": [
                "EventDate",
                "EndDate",
                "Title"
            ]
        },
        "ViewTypeKind": 1,
        "ViewType2": "MODERNCALENDAR",
        "ViewData": "<FieldRef Name=\"Title\" Type=\"CalendarMonthTitle\" /><FieldRef Name=\"Title\" Type=\"CalendarWeekTitle\" /><FieldRef Name=\"Title\" Type=\"CalendarWeekLocation\" /><FieldRef Name=\"Title\" Type=\"CalendarDayTitle\" /><FieldRef Name=\"Title\" Type=\"CalendarDayLocation\" />",
        "CalendarViewStyles": "<CalendarViewStyle Title=\"Day\" Type=\"day\" Template=\"CalendarViewdayChrome\" Sequence=\"1\" Default=\"FALSE\" /><CalendarViewStyle Title=\"Week\" Type=\"week\" Template=\"CalendarViewweekChrome\" Sequence=\"2\" Default=\"FALSE\" /><CalendarViewStyle Title=\"Month\" Type=\"month\" Template=\"CalendarViewmonthChrome\" Sequence=\"3\" Default=\"TRUE\" />",
        "Query": "",
        "Paged": true,
        "PersonalView": false,
        "RowLimit": 0
    }
}
"@

Invoke-PnPSPRestMethod -Method Post -Url "$url/_api/web/lists/GetByTitle('$listname')/Views/Add" -ContentType "application/json;odata=verbose" -Content $viewCreationJson

#Optional Commands
Set-PnPList -Identity $listname -ListExperience NewExperience # Set list experience to force the list to display in Modern
Set-PnPView -List $listname -Identity $newViewTitle -Values @{DefaultView=$true;MobileView=$true;MobileDefaultView=$true} #Set newly created view To Be Default
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)(/.*)?$')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory, HelpMessage = "Title of the list to add the calendar view to")]
    [string]$ListTitle,
    
    [Parameter(Mandatory, HelpMessage = "Title of the new calendar view")]
    [string]$ViewTitle,
    
    [Parameter(Mandatory, HelpMessage = "Internal name of the field containing the start date")]
    [string]$StartDateField,
    
    [Parameter(Mandatory, HelpMessage = "Internal name of the field containing the end date")]
    [string]$EndDateField,
    
    [Parameter(HelpMessage = "Internal name of the field to use as event title (default: Title)")]
    [string]$TitleField = "Title",
    
    [Parameter(HelpMessage = "Internal name of the field to use as event subtitle/location")]
    [string]$SubTitleField,
    
    [Parameter(HelpMessage = "Comma-separated list of additional fields to display in the view")]
    [string]$AdditionalFields,
    
    [Parameter(HelpMessage = "Default layout for calendar view (month, week, workWeek, day)")]
    [ValidateSet('month', 'week', 'workWeek', 'day')]
    [string]$DefaultLayout = 'month',
    
    [Parameter(HelpMessage = "Set the created view as the default view")]
    [switch]$SetAsDefault,
    
    [Parameter(HelpMessage = "Set the list experience to modern/new experience")]
    [switch]$SetListExperience
)

begin {
    $transcriptPath = "$((Get-Location).Path)/AddModernCalendarView-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Add Modern Calendar View - CLI for Microsoft 365" -ForegroundColor Cyan
    Write-Host "===============================================" -ForegroundColor Cyan
    Write-Host ""
    
    Write-Verbose "Ensuring Microsoft 365 login..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to ensure Microsoft 365 login. Please run 'm365 login' first."
    }
    Write-Host "✓ Authenticated successfully" -ForegroundColor Green
    Write-Host ""
    
    Write-Host "Site URL: $SiteUrl" -ForegroundColor White
    Write-Host "List: $ListTitle" -ForegroundColor White
    Write-Host "View Title: $ViewTitle" -ForegroundColor White
    Write-Host "Start Date Field: $StartDateField" -ForegroundColor White
    Write-Host "End Date Field: $EndDateField" -ForegroundColor White
    Write-Host "Title Field: $TitleField" -ForegroundColor White
    if ($SubTitleField) {
        Write-Host "SubTitle Field: $SubTitleField" -ForegroundColor White
    }
    Write-Host "Default Layout: $DefaultLayout" -ForegroundColor White
    Write-Host ""
}

process {
   try {
       if ($SetListExperience) {
           Write-Host "Setting list experience to modern..." -ForegroundColor Cyan
           
           if ($PSCmdlet.ShouldProcess($ListTitle, "Set list experience to modern")) {
               $setListResult = m365 spo list set --webUrl $SiteUrl --title $ListTitle --listExperienceOptions NewExperience 2>&1
               
               if ($LASTEXITCODE -ne 0) {
                   Write-Warning "Failed to set list experience: $setListResult"
               }
               else {
                   Write-Host "✓ Successfully set list experience to modern" -ForegroundColor Green
               }
           }
           Write-Host ""
       }
       
       Write-Host "Creating modern calendar view..." -ForegroundColor Cyan
        
        $viewFields = @($TitleField)
        if ($AdditionalFields) {
            $viewFields += $AdditionalFields.Split(',').Trim()
        }
        $fieldsParam = $viewFields -join ','
        
        $action = "Create calendar view '$ViewTitle'"
        if ($PSCmdlet.ShouldProcess($ListTitle, $action)) {
            $args = @(
                'spo', 'list', 'view', 'add',
                '--webUrl', $SiteUrl,
                '--listTitle', $ListTitle,
                '--title', $ViewTitle,
                '--type', 'calendar',
                '--calendarStartDateField', $StartDateField,
                '--calendarEndDateField', $EndDateField,
                '--calendarTitleField', $TitleField,
                '--calendarDefaultLayout', $DefaultLayout,
                '--fields', $fieldsParam,
                '--rowLimit', '0',
                '--output', 'json'
            )
            
            if ($SubTitleField) {
                $args += '--calendarSubTitleField'
                $args += $SubTitleField
            }
            
            if ($SetAsDefault) {
                $args += '--default'
            }
            
            Write-Verbose "Executing: m365 $($args -join ' ')"
            $viewJson = m365 @args 2>&1
            
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to create calendar view: $viewJson"
            }
            
           $view = $viewJson | ConvertFrom-Json
           Write-Host "✓ Successfully created calendar view: $($view.Title)" -ForegroundColor Green
           Write-Verbose "View ID: $($view.Id)"
       }
       else {
            Write-Host "WhatIf: Would create calendar view '$ViewTitle' in list '$ListTitle'" -ForegroundColor Cyan
        }
    }
    catch {
        Write-Host "✗ Error: $_" -ForegroundColor Red
        throw
    }
}

end {
    Write-Host ""
    Write-Host "===================" -ForegroundColor Cyan
    Write-Host "Operation Complete" -ForegroundColor Cyan
    Write-Host "===================" -ForegroundColor Cyan
    Write-Host ""
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Gray
    
    Stop-Transcript
}

# Example 1: Create a basic month view calendar
# .\Add-ModernCalendarView.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Events" -ViewTitle "Calendar" -StartDateField "EventDate" -EndDateField "EndDate"

# Example 2: Create a week view calendar with subtitle and set as default
# .\Add-ModernCalendarView.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Events" -ViewTitle "Weekly Calendar" -StartDateField "EventDate" -EndDateField "EndDate" -SubTitleField "Location" -DefaultLayout week -SetAsDefault

# Example 3: Create calendar with additional fields and modern list experience
# .\Add-ModernCalendarView.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Events" -ViewTitle "Full Calendar" -StartDateField "EventDate" -EndDateField "EndDate" -AdditionalFields "Category,Organizer" -SetListExperience

# Example 4: Test with WhatIf before creating
# .\Add-ModernCalendarView.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/project" -ListTitle "Events" -ViewTitle "Calendar" -StartDateField "EventDate" -EndDateField "EndDate" -WhatIf
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

* Sample first appeared on [Adding the New Modern Calendar View to a SharePoint List using PnP PowerShell - Leon Armston Blog](https://www.leonarmston.com/2021/11/adding-the-new-modern-calendar-view-to-a-sharepoint-list-using-pnp-powershell/)
* JSON body explanation [stackoverflow](https://stackoverflow.com/questions/67271425/create-modern-calendar-view-for-sharepoint-online-list-using-the-rest-api) - credit [@shagra-ms](https://github.com/shagra-ms)


## Contributors

| Author(s) |
|-----------|
| Adam Wójcik |
| [Leon Armston](https://github.com/LeonArmston) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-add-modern-calendar-view" aria-hidden="true" />

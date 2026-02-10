

# Export CSV To SharePoint List Data

## Summary

This script demonstrates how to create a SharePoint list with custom fields and bulk import employee data from a CSV file using CLI for Microsoft 365 v11.4.0+. It creates a list with 6 custom fields (4 text fields and 2 datetime fields) and imports all records from a CSV file. The script includes validation, error handling, and progress tracking.

**Note**: DateTime fields require the format that matches your site's regional settings. A universal format that works on all regions is: `yyyy-MM-dd HH:mm:ss` (e.g., `2020-03-01 09:00:00`).

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as below,
- First, we will connect the site URL with the user's credentials.
    - To connect the SharePoint site with PnP refer to this article.
    - Then we will create a list and fields. so field types will be as a below,
    - FirstName,LastName,JobTitle,Location - Single line of text
    - BirthDate, HireDate - Date and time

We will import the CSV using the Import-Csv method.

# [PnP PowerShell](#tab/pnpps)
```powershell

$Login = #userid    
$password = #password  
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force     
$Creds = New-Object -Typename System.Management.Automation.PSCredential -ArgumentList $Login, $secureStringPwd   
$siteUrl = #siteUrl  
 
#connect to site  
Write-Host "Connection to the site..." -ForegroundColor Yellow  
Connect-PnpOnline -Url $SiteUrl -Credentials $Creds       
Write-Host "Connection successfully..." -ForegroundColor Yellow  
 
#create a list  
Write-Host "Creating list..." -ForegroundColor Yellow  
New-PnPList -Title "Employees" -Url "lists/Employees"   
Write-Host "List created..." -ForegroundColor Yellow  
 
#create fields  
Write-Host "Creating fields..." -ForegroundColor Yellow  
Add-PnPField -List "Employees" -DisplayName "First Name" -InternalName "FirstName" -Type Text -AddToDefaultView  
Add-PnPField -List "Employees" -DisplayName "Last Name" -InternalName "LastName" -Type Text -AddToDefaultView  
Add-PnPField -List "Employees" -DisplayName "Location" -InternalName "Location" -Type Text -AddToDefaultView  
Add-PnPField -List "Employees" -DisplayName "Job Title" -InternalName "JobTitle" -Type Text -AddToDefaultView   
Add-PnPField -List "Employees" -DisplayName "Hire Date" -InternalName "HireDate" -Type DateTime -AddToDefaultView  
Add-PnPField -List "Employees" -DisplayName "Birth Date" -InternalName "BirthDate" -Type DateTime -AddToDefaultView  
Write-Host "Fields created..." -ForegroundColor Yellow  
  
$filePath = "F:\Intranet Employee Report.csv"  
 
#Import CSV  
$CSVRecords = Import-Csv $FilePath  
Write-host -f Yellow "$($CSVRecords.count) Rows Found!"  
 
#create list items  
Write-Host "Creating list items..." -ForegroundColor Yellow  
foreach ($Record in $CSVRecords) {  
    $items = Add-PnPListItem -List "Employees" -Values @{  
        "Title"     = $Record.'FirstName' + " " + $Record.'LastName';  
        "FirstName" = $Record.'FirstName';  
        "LastName"  = $Record.'LastName';  
        "Location"  = $Record.'Location';  
        "JobTitle"  = $Record.'JobTitle';        
        "BirthDate" = $Record.'BirthDate';  
        "HireDate"  = $Record.'HireDate';  
    }  
}  
  
Write-Host "list items created..." -ForegroundColor Yellow  

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$SiteUrl,

    [Parameter(HelpMessage = "Title of the list to create")]
    [ValidateNotNullOrEmpty()]
    [string]$ListTitle = "Employees",

    [Parameter(Mandatory, HelpMessage = "Path to the CSV file containing employee data")]
    [ValidateScript({
        if (Test-Path $_ -PathType Leaf) { $true }
        else { throw "CSV file not found: $_" }
    })]
    [string]$CsvPath,

    [Parameter(HelpMessage = "Add fields to default view")]
    [switch]$AddToDefaultView = $true
)

begin {
    $transcriptPath = Join-Path (Get-Location).Path "transcript_$(Get-Date -Format 'yyyyMMddHHmmss').log"
    Start-Transcript -Path $transcriptPath

    $script:Summary = [PSCustomObject]@{
        TotalRecordsInCsv = 0
        ItemsCreated      = 0
        ItemsFailed       = 0
    }

    Write-Host "Authenticating to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate to Microsoft 365"
    }
    Write-Host "Authentication successful" -ForegroundColor Green

    Write-Host "Validating CSV file..." -ForegroundColor Cyan
    $csvData = Import-Csv -Path $CsvPath
    $requiredFields = @('FirstName', 'LastName', 'JobTitle', 'Location', 'BirthDate', 'HireDate')
    $csvHeaders = $csvData[0].PSObject.Properties.Name
    $missingFields = $requiredFields | Where-Object { $_ -notin $csvHeaders }
    if ($missingFields) {
        throw "CSV file is missing required fields: $($missingFields -join ', '). Required fields: $($requiredFields -join ', ')"
    }
    $script:Summary.TotalRecordsInCsv = $csvData.Count
    Write-Host "CSV validation successful. Found $($csvData.Count) records" -ForegroundColor Green

    Write-Host "Creating list '$ListTitle'..." -ForegroundColor Cyan
    try {
        m365 spo list add --title $ListTitle --baseTemplate GenericList --webUrl $SiteUrl --output json | Out-Null
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to create list"
        }
        Write-Host "List created successfully" -ForegroundColor Green
    }
    catch {
        Stop-Transcript
        throw "Failed to create list '$ListTitle': $($_.Exception.Message)"
    }

    Write-Host "Creating custom fields..." -ForegroundColor Cyan
    $fieldDefinitions = @(
        @{ Name = 'FirstName'; DisplayName = 'FirstName'; Type = 'Text'; Id = '{6085e32a-339b-4da7-ab6d-c1e013e5ab27}' },
        @{ Name = 'LastName'; DisplayName = 'LastName'; Type = 'Text'; Id = '{1b9be491-0a09-4381-b9e2-7a980a5b8ad9}' },
        @{ Name = 'Location'; DisplayName = 'Location'; Type = 'Text'; Id = '{b801e08f-c9e1-406d-a044-237f576157be}' },
        @{ Name = 'JobTitle'; DisplayName = 'JobTitle'; Type = 'Text'; Id = '{127da56f-8d7f-4f36-a461-afab9f5c6f34}' },
        @{ Name = 'HireDate'; DisplayName = 'HireDate'; Type = 'DateTime'; Id = '{41351989-e693-430d-9c40-d4e19c47df08}' },
        @{ Name = 'BirthDate'; DisplayName = 'BirthDate'; Type = 'DateTime'; Id = '{b0541eb4-d16f-4b44-a92a-d36a2e3f88ba}' }
    )

    foreach ($field in $fieldDefinitions) {
        try {
            $fieldXml = "<Field Type='$($field.Type)' DisplayName='$($field.DisplayName)' Required='FALSE' EnforceUniqueValues='FALSE' Indexed='FALSE' ID='$($field.Id)' SourceID='{4f118c69-66e0-497c-96ff-d7855ce0713d}' StaticName='$($field.Name)' Name='$($field.Name)'></Field>"
            m365 spo field add --webUrl $SiteUrl --listTitle $ListTitle --xml $fieldXml --output json | Out-Null
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to create field $($field.Name)"
            }

            if ($AddToDefaultView) {
                m365 spo list view field add --webUrl $SiteUrl --listTitle $ListTitle --viewTitle 'All Items' --title $field.Name --output json | Out-Null
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to add field $($field.Name) to default view"
                }
            }
        }
        catch {
            Stop-Transcript
            throw "Failed to create field '$($field.Name)': $($_.Exception.Message)"
        }
    }
    Write-Host "Created $($fieldDefinitions.Count) custom fields successfully" -ForegroundColor Green
}

process {
    $csvRecords = Import-Csv -Path $CsvPath
    Write-Host "Importing $($csvRecords.Count) records from CSV..." -ForegroundColor Cyan

    $counter = 0
    foreach ($record in $csvRecords) {
        $counter++
        $percentComplete = [math]::Round(($counter / $csvRecords.Count) * 100, 0)
        Write-Progress -Activity "Importing employee records" -Status "Processing record $counter of $($csvRecords.Count)" -PercentComplete $percentComplete

        $title = "$($record.FirstName) $($record.LastName)"
        if ($PSCmdlet.ShouldProcess($title, 'Add list item')) {
            try {
                m365 spo listitem add --listTitle $ListTitle --webUrl $SiteUrl --Title $title --FirstName $record.FirstName --LastName $record.LastName --Location $record.Location --JobTitle $record.JobTitle --BirthDate $record.BirthDate --HireDate $record.HireDate --output json | Out-Null
                if ($LASTEXITCODE -ne 0) {
                    throw "CLI command failed with exit code $LASTEXITCODE"
                }
                $script:Summary.ItemsCreated++
            }
            catch {
                Write-Warning "Failed to add item for $title: $($_.Exception.Message)"
                $script:Summary.ItemsFailed++
                continue
            }
        }
    }
    Write-Progress -Activity "Importing employee records" -Completed
}

end {
    Write-Host "\n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Site URL: $SiteUrl" -ForegroundColor White
    Write-Host "List Title: $ListTitle" -ForegroundColor White
    Write-Host "Total Records in CSV: $($script:Summary.TotalRecordsInCsv)" -ForegroundColor White
    Write-Host "Items Created: $($script:Summary.ItemsCreated)" -ForegroundColor Green
    if ($script:Summary.ItemsFailed -gt 0) {
        Write-Host "Items Failed: $($script:Summary.ItemsFailed)" -ForegroundColor Red
    } else {
        Write-Host "Items Failed: $($script:Summary.ItemsFailed)" -ForegroundColor Green
    }
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Cyan
    Write-Host "============================" -ForegroundColor Cyan

    Stop-Transcript
}

# Example 1: Test with WhatIf (safe mode, no changes made)
# .\\Import-CsvToSharePointList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -CsvPath "C:\\Data\\employees.csv" -WhatIf

# Example 2: Basic usage
# .\\Import-CsvToSharePointList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -CsvPath "C:\\Data\\employees.csv"

# Example 3: Custom list title
# .\\Import-CsvToSharePointList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -ListTitle "Staff Members" -CsvPath "C:\\Data\\employees.csv"

# Example 4: Verbose mode with custom list and no default view
# .\\Import-CsvToSharePointList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/hr" -ListTitle "Personnel" -CsvPath "C:\\Data\\employees.csv" -AddToDefaultView:$false -Verbose

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Source Credit

Sample first appeared on [https://www.c-sharpcorner.com/article/export-csv-to-sharepoint-list-data-using-pnp-powershell/](https://www.c-sharpcorner.com/article/export-csv-to-sharepoint-list-data-using-pnp-powershell/)

## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati |
| [Adam Wójcik](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-data-to-sharepoint-lists" aria-hidden="true" />

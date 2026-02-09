

# List custom fields from SharePoint Lists or libraries

## Summary

This script generates a CSV report of all custom columns/fields in SharePoint lists or libraries using CLI for Microsoft 365 (v11.4.0+), excluding out-of-the-box system fields and lists.

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as given below
- Update the `$siteUrl`, `$ReportOutput` and optionally update `$SystemFlds` & `$SystemLists` to remove any values you would like to include in the report

# [PnP PowerShell](#tab/pnpps)

```powershell
# Connect to SharePoint site
$siteUrl = "https://contoso.sharepoint.com/teams/d-app"
Connect-PnPOnline -Url $siteUrl -Interactive

$dateTime = (Get-Date).toString("dd-MM-yyyy")
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$ReportOutput = "FieldsReports-" + $dateTime + ".csv"

$OutPutFieldsFile = $directorypath + "\Logs\"+ $ReportOutput

#Arry to Skip OOB Fields
$SystemFlds = @(
    "Compliance Asset Id","Body","Expires","ID","Content Type","Modified","Created","Created By","Modified By","Version","Attachments","Edit","Type","Item Child Count","Folder", "Child Count","App Created By","App Modified By", "Name","Checked Out To","Check In Comment","File Size","Source Version (Converted Document)","Source Name (Converted Document)","Location","Start Time","End Time","Description","All Day Event","Recurrence","Attendees","Category","Resources","Free/Busy","Check Double Booking","Enterprise Keywords", "Last Updated","Parent Item Editor","Parent Item ID","Last Reply By","Question","Best Response","Best Response Id", "Is Featured Discussion","E-Mail Sender","Replies","Folder Child Count","Discussion Subject","Reply","Post","Threading","Posted By", "Due Date","Assigned To","File Received","Number Of Setups","Notes/Comments","Task_Status","Is Approval Required","Approver","Approver Comments","Approval Date","Documents", "Order","Role","Person or Group","Location", "Predecessors","Priority","Task Status","% Complete","Start Date","Completed","Related Items", "Background Image Location","Link Location","Launch Behavior","Background Image Cluster Horizontal Start","Background Image Cluster Vertical Start", "First Name","Full Name","Email Address","Company","Job Title","Business Phone","Home Phone","Mobile Number","Fax Number","Address","City","State/Province","ZIP/Postal Code","Country/Region","Web Page","Notes","Name","Order","Role", "Color Tag", "Label setting", "Retention label", "Retention Label Applied", "Label applied by", "Item is a Record" ,"Comment Count","Like Count","Sensitivity", "Copy Source","Title"
)

#Arry to Skip System Lists and Libraries
$SystemLists = @(
    "Converted Forms", "Master Page Gallery", "Customized Reports", "Form Templates", "List Template Gallery", "Theme Gallery", "Apps for SharePoint", "Reporting Templates", "Solution Gallery", "Style Library", "Web Part Gallery", "Site Assets", "wfpub", "Site Pages", "Images", "MicroFeed", "Pages"
)

#Get all lists from the site
$FieldsCollection = @()
$lists = Get-PnPList | Where {$_.Hidden -eq $false -and $SystemLists -notcontains $_.Title } | ForEach-Object {
    $list = $_.Title
    Get-PnPField -List $list | Where {$_.Hidden -eq $false -and $SystemFlds -notcontains $_.Title } | ForEach-Object {
        $ExportField = New-Object PSObject
        $ExportField | Add-Member -MemberType NoteProperty -name "List" -value $list
        $ExportField | Add-Member -MemberType NoteProperty -name "FieldName" -value $_.Title
        $FieldsCollection += $ExportField
    }
}

#Export to CSV
$FieldsCollection | Export-Csv -Path $OutPutFieldsFile -NoTypeInformation 

#Disconnect from SharePoint site
Disconnect-PnPOnline
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)$')]
    [string]$SiteUrl,
    
    [Parameter(HelpMessage = "Path where CSV report will be saved")]
    [ValidateNotNullOrEmpty()]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "custom-fields-report-$timestamp.log"
    $csvPath = Join-Path $OutputPath "custom-fields-$timestamp.csv"
    
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Yellow
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to authenticate to Microsoft 365"
    }
    
    $script:Summary = @{
        TotalLists = 0
        TotalFields = 0
    }
    
    $script:ReportCollection = @()
    
    $SystemFlds = @(
        "Compliance Asset Id","Body","Expires","ID","Content Type","Modified","Created","Created By","Modified By","Version","Attachments","Edit","Type","Item Child Count","Folder", "Child Count","App Created By","App Modified By", "Name","Checked Out To","Check In Comment","File Size","Source Version (Converted Document)","Source Name (Converted Document)","Location","Start Time","End Time","Description","All Day Event","Recurrence","Attendees","Category","Resources","Free/Busy","Check Double Booking","Enterprise Keywords", "Last Updated","Parent Item Editor","Parent Item ID","Last Reply By","Question","Best Response","Best Response Id", "Is Featured Discussion","E-Mail Sender","Replies","Folder Child Count","Discussion Subject","Reply","Post","Threading","Posted By", "Due Date","Assigned To","File Received","Number Of Setups","Notes/Comments","Task_Status","Is Approval Required","Approver","Approver Comments","Approval Date","Documents", "Order","Role","Person or Group","Location", "Predecessors","Priority","Task Status","% Complete","Start Date","Completed","Related Items", "Background Image Location","Link Location","Launch Behavior","Background Image Cluster Horizontal Start","Background Image Cluster Vertical Start", "First Name","Full Name","Email Address","Company","Job Title","Business Phone","Home Phone","Mobile Number","Fax Number","Address","City","State/Province","ZIP/Postal Code","Country/Region","Web Page","Notes","Name","Order","Role", "Color Tag", "Label setting", "Retention label", "Retention Label Applied", "Label applied by", "Item is a Record" ,"Comment Count","Like Count","Sensitivity", "Copy Source","Title"
    )
    
    $SystemLists = @(
        "Converted Forms", "Master Page Gallery", "Customized Reports", "Form Templates", "List Template Gallery", "Theme Gallery", "Apps for SharePoint", "Reporting Templates", "Solution Gallery", "Style Library", "Web Part Gallery", "Site Assets", "wfpub", "Site Pages", "Images", "MicroFeed", "Pages"
    )
}

process {
    Write-Host "Retrieving all lists from site: $SiteUrl" -ForegroundColor Yellow
    
    try {
        $lists = m365 spo list list --webUrl $SiteUrl --output json | ConvertFrom-Json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve lists"
        }
        
        $filteredLists = @($lists | Where-Object { $_.Hidden -eq $false -and $SystemLists -notcontains $_.Title })
        $script:Summary.TotalLists = $filteredLists.Count
        
        Write-Host "Found $($filteredLists.Count) non-system lists" -ForegroundColor Green
        
        $listIndex = 0
        foreach ($listItem in $filteredLists) {
            $listIndex++
            $list = $listItem.Title
            
            Write-Host "[$listIndex/$($filteredLists.Count)] Processing list: $list" -ForegroundColor Cyan
            
            try {
                $fields = m365 spo field list --webUrl $SiteUrl --listTitle $list --output json | ConvertFrom-Json
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve fields for list '$list'"
                    continue
                }
                
                $customFields = @($fields | Where-Object { $_.Hidden -eq $false -and $SystemFlds -notcontains $_.Title })
                
                foreach ($field in $customFields) {
                    $ExportField = [PSCustomObject]@{
                        List = $list
                        FieldName = $field.Title
                    }
                    $script:ReportCollection += $ExportField
                    $script:Summary.TotalFields++
                }
                
                Write-Host "  Found $($customFields.Count) custom fields" -ForegroundColor Gray
            }
            catch {
                Write-Warning "Error processing list '$list': $($_.Exception.Message)"
                continue
            }
        }
    }
    catch {
        Write-Error "Failed to retrieve lists: $($_.Exception.Message)"
        Stop-Transcript
        throw
    }
}

end {
    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Site URL: $SiteUrl" -ForegroundColor White
    Write-Host "Total Lists Processed: $($Summary.TotalLists)" -ForegroundColor Green
    Write-Host "Total Custom Fields Found: $($Summary.TotalFields)" -ForegroundColor Green
    
    if ($script:ReportCollection.Count -gt 0) {
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation
        Write-Host "CSV report saved to: $csvPath" -ForegroundColor Cyan
    } else {
        Write-Host "No custom fields found" -ForegroundColor Yellow
    }
    
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Cyan
    Stop-Transcript
}

# Example 1: Basic usage
# .\Get-CustomFields.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/spconnect"

# Example 2: Custom output path
# .\Get-CustomFields.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/spconnect" -OutputPath "C:\Reports"

# Example 3: Verbose mode
# .\Get-CustomFields.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/spconnect" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Reshmee Auckloo](https://github.com/reshmee011)|
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-customfields-lists" aria-hidden="true" />



# Get SharePoint List Fields With Required properties And Export It To CSV

## Summary

This sample read site url and list/library title from user and fetch some required field properties and then export it to CSV

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as below,
- First, we will Read site URL from user and connect to the Site.
	- then we will Read list/library title from user and get fields information.
    - And then we will export it to CSV with some required properties.
 
# [PnP PowerShell](#tab/pnpps)
```powershell

$username = "chandani@domain.onmicrosoft.com"
$password = "*******"
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$Creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd
$global:listFields = @()
$BasePath = "E:\Contribution\PnP-Scripts\ListFields\"
$DateTime = "{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$CSVPath = $BasePath + "\listfields" + $DateTime + ".csv"

Function ConnectToSPSite() {
    try {
        $SiteUrl = Read-Host "Please enter Site URL"
        if ($SiteUrl) {
            Write-Host "Connecting to Site :'$($SiteUrl)'..." -ForegroundColor Yellow  
            Connect-PnPOnline -Url $SiteUrl -Credentials $Creds
            Write-Host "Connection Successfull to site: '$($SiteUrl)'" -ForegroundColor Green              
            GetListFields
        }
        else {
            Write-Host "Source Site URL is empty." -ForegroundColor Red
        }
    }
    catch {
        Write-Host "Error in connecting to Site:'$($SiteUrl)'" $_.Exception.Message -ForegroundColor Red               
    } 
}

Function GetListFields() {
    try {
        $ListName =  Read-Host "Please enter list name"
        if ($ListName) {
            Write-Host "Getting fields from :'$($ListName)'..." -ForegroundColor Yellow  
            $ListFields = Get-PnPField -List $ListName
            Write-Host "Getting fields from :'$($ListName)' Successfully!" -ForegroundColor Green  
            foreach ($ListField in $ListFields) {  
                $global:listFields += New-Object PSObject -Property ([ordered]@{
                        "Title"            = $ListField.Title                           
                        "Type"             = $ListField.TypeAsString                         
                        "Internal Name"    = $ListField.InternalName  
                        "Static Name"      = $ListField.StaticName  
                        "Scope"            = $ListField.Scope  
                        "Type DisplayName" = $ListField.TypeDisplayName                          
                        "Is read only?"    = $ListField.ReadOnlyField  
                        "Unique?"          = $ListField.EnforceUniqueValues  
                        "IsRequired"       = $ListField.Required
                        "IsSortable"       = $ListField.Sortable
                        "Schema XML"       = $ListField.SchemaXml
                        "Description"      = $ListField.Description 
                        "Group Name"       = $ListField.Group   
                    })
            }  
        }
        else {
            Write-Host "List name is empty." -ForegroundColor Red
        }
        BindingtoCSV($global:listFields)
        $global:listFields = @()
    }
    catch {
        Write-Host "Error in getting list fields from :'$($ListName)'" $_.Exception.Message -ForegroundColor Red               
    } 
    Write-Host "Export to CSV Successfully!" -ForegroundColor Green
}

Function BindingtoCSV {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $Global)       
    $global:listFields | Export-Csv $CSVPath -NoTypeInformation -Append            
}

ConnectToSPSite

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]


# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
function Get-SpoListFieldReport {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL hosting the list")]
        [ValidateNotNullOrEmpty()]
        [string] $SiteUrl,

        [Parameter(Mandatory = $true, HelpMessage = "Title of the target list or library")]
        [ValidateNotNullOrEmpty()]
        [string] $ListTitle,

        [Parameter(HelpMessage = "When supplied, include hidden fields in the report")]
        [switch] $IncludeHidden,

        [Parameter(HelpMessage = "Skip exporting the report to disk")]
        [switch] $SkipExport,

        [Parameter(HelpMessage = "Custom CSV destination (defaults to timestamped file in the current directory)")]
        [ValidateNotNullOrEmpty()]
        [string] $OutputPath,

        [Parameter(HelpMessage = "Emit the report objects to the pipeline")]
        [switch] $PassThru
    )

    begin {
        Write-Verbose "Ensuring CLI authentication"
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to sign in to CLI for Microsoft 365. CLI output: $loginOutput"
        }

        if (-not $SkipExport) {
            if (-not $PSBoundParameters.ContainsKey('OutputPath')) {
                $timestamp = (Get-Date).ToString('yyyyMMdd-HHmmss')
                $OutputPath = Join-Path -Path (Get-Location) -ChildPath "ListFields-$timestamp.csv"
            }

            $directory = Split-Path -Parent $OutputPath
            if ([string]::IsNullOrEmpty($directory)) {
                $directory = '.'
            }

            if (-not (Test-Path -Path $directory)) {
                Write-Verbose "Creating directory '$directory'"
                New-Item -Path $directory -ItemType Directory -Force | Out-Null
            }
        }

        $context = [ordered]@{
            Results = [System.Collections.Generic.List[psobject]]::new()
            FieldsRetrieved = 0
            HiddenSkipped   = 0
            Exported        = 0
        }

        $query = "[].{Id:Id,Title:Title,Type:TypeAsString,InternalName:InternalName,StaticName:StaticName,Scope:Scope,TypeDisplay:TypeDisplayName,ReadOnly:ReadOnlyField,Unique:EnforceUniqueValues,Required:Required,Sortable:Sortable,SchemaXml:SchemaXml,Description:Description,Group:Group,Hidden:Hidden}"
        $properties = 'Id,Title,TypeAsString,InternalName,StaticName,Scope,TypeDisplayName,ReadOnlyField,EnforceUniqueValues,Required,Sortable,SchemaXml,Description,Group,Hidden'
    }

    process {
        Write-Verbose "Retrieving fields for list '$ListTitle'"
        $fieldsJson = m365 spo field list --webUrl $SiteUrl --listTitle $ListTitle --properties $properties --output json --query $query 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list fields. CLI output: $fieldsJson"
        }

        try {
            $fields = if ([string]::IsNullOrWhiteSpace($fieldsJson)) { @() } else { @($fieldsJson | ConvertFrom-Json) }
        }
        catch {
            throw "Failed to parse field listing. Error: $($_.Exception.Message)"
        }

        $context.FieldsRetrieved = $fields.Count

        if (-not $IncludeHidden) {
            $visibleFields = $fields | Where-Object { -not $_.Hidden }
            $context.HiddenSkipped = $fields.Count - $visibleFields.Count
        }
        else {
            $visibleFields = $fields
        }

        foreach ($field in $visibleFields) {
            $context.Results.Add([pscustomobject]@{
                Title        = $field.Title
                Type         = $field.Type
                InternalName = $field.InternalName
                StaticName   = $field.StaticName
                Scope        = $field.Scope
                TypeDisplay  = $field.TypeDisplay
                ReadOnly     = [bool]$field.ReadOnly
                Unique       = [bool]$field.Unique
                Required     = [bool]$field.Required
                Sortable     = [bool]$field.Sortable
                Group        = $field.Group
                Hidden       = [bool]$field.Hidden
                Description  = $field.Description
                SchemaXml    = $field.SchemaXml
            }) | Out-Null
        }
    }

    end {
        if (-not $SkipExport -and $context.Results.Count -gt 0) {
            Write-Verbose "Exporting field report to '$OutputPath'"
            $context.Results | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8 -Force
            $context.Exported = $context.Results.Count
        }

        Write-Host "Field export summary:" -ForegroundColor Cyan
        Write-Host ("- Fields retrieved : {0}" -f $context.FieldsRetrieved)
        Write-Host ("- Hidden skipped   : {0}" -f $context.HiddenSkipped)
        Write-Host ("- Rows in report   : {0}" -f $context.Results.Count)
        Write-Host ("- Rows exported    : {0}" -f $context.Exported)

        if ($PassThru) {
            $context.Results
        }
    }
}

# Example usage
Get-SpoListFieldReport -SiteUrl "https://contoso.sharepoint.com/sites/demo" -ListTitle "Documents" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati |
| Nanddeep Nachan |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-and-export-list-fields" aria-hidden="true" />

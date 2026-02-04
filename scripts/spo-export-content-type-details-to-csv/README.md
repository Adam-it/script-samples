

# Export Content Type Details To CSV

## Summary

This example illustrates how to export all content types present on a SharePoint site, capturing essential details like Name, ID, Scope, Schema, Fields, and additional information, then organizing them into a CSV format.

# [PnP PowerShell](#tab/pnpps)

```powershell
$siteUrl = Read-Host "Enter site URL"
$username = "username@domain.onmicrosoft.com"
$password = "********"
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd
$dateTime = "{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$basePath = "D:\Contributions\Scripts\Logs\"
$csvPath = $basePath + "\ContentTypeData" + $dateTime + ".csv"
$global:ctData = @()

Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $creds)
    Write-Host "Connecting to Site '$($siteUrl)'" -ForegroundColor Yellow
    Connect-PnPOnline -Url $siteUrl -Credentials $creds
    Write-Host "Connection Successful!" -ForegroundColor Green
}

Function ContentTypeDetails() {
    try {
        Write-Host "Getting content type details..." -ForegroundColor Yellow
        $allContentTypes = Get-PnPContentType
        Foreach ($contentType in $allContentTypes)
        {
            #Collect Content Type Data
            $ctName = $contentType.Name
            $ctId = $contentType.Id
            $ctGroup = $contentType.Group
            $ctDescription = $contentType.Description
            $ctPath = $contentType.Path
            $ctScope = $contentType.Scope
            $ctStringId = $contentType.StringId
            $ctSchemaXml = $contentType.SchemaXml
            $contentTypeFields = Get-PnPProperty -ClientObject $contentType -Property Fields
            $contentTypeFieldsCount = $contentTypeFields.Count
            $contentTypeFieldsSchema = $contentTypeFields.SchemaXml
            $contentTypeTitle = ($contentTypeFields | select-object -property Title | foreach-object { $_.Title }) -join ','
            
            $global:ctData += [PSCustomObject] @{
                Name                = $ctName
                ID                  = $ctId
                Group               = $ctGroup
                Description         = $ctDescription
                Path                = $ctPath
                Scope               = $ctScope
                StringId            = $ctStringId
                SchemaXml           = $ctSchemaXml
                Fields              = $contentTypeTitle
                FieldCount          = $contentTypeFieldsCount
                FieldSchemaXMl      = $contentTypeFields.SchemaXml
            }
        }
        Write-Host "Getting content type details successfully!..." -ForegroundColor Green
    }
    catch {
        Write-Host "Error in getting content type information:" $_.Exception.Message -ForegroundColor Red
    }
    Write-Host "Exporting to CSV..."  -ForegroundColor Yellow
    $global:ctData | Export-Csv $csvPath -NoTypeInformation -Append
    Write-Host "Exported to CSV successfully!..."  -ForegroundColor Green

    # Disconnect SharePoint online connection
    Disconnect-PnPOnline
}

Function StartProcessing {
    Login($creds);
    ContentTypeDetails
}

StartProcessing
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL (e.g., https://contoso.sharepoint.com/sites/marketing)")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn).*$')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Output directory path for the CSV file")]
    [ValidateScript({
        if ($PSBoundParameters.ContainsKey('OutputPath') -and -not (Test-Path -Path $_ -PathType Container)) {
            throw "The directory '$_' does not exist."
        }
        $true
    })]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    Write-Verbose "Ensuring authentication to Microsoft 365..."
    m365 login --ensure
    
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with Microsoft 365. Please try again."
    }
    
    Write-Verbose "Authentication successful."
    
    $script:ContentTypeCollection = @()
    
    $timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
    $csvFileName = "ContentTypes_$timestamp.csv"
    $csvPath = Join-Path -Path $OutputPath -ChildPath $csvFileName
    
    $transcriptPath = Join-Path -Path $OutputPath -ChildPath "ContentTypeExport_$timestamp.log"
    Start-Transcript -Path $transcriptPath | Out-Null
    
    Write-Host "Starting content type export from: $SiteUrl" -ForegroundColor Cyan
}

process {
    try {
        Write-Verbose "Fetching content types from site..."
        
        $result = m365 spo contenttype list --webUrl $SiteUrl --output json
        
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve content types from site: $SiteUrl"
        }
        
        $contentTypes = @($result | ConvertFrom-Json)
        Write-Verbose "Found $($contentTypes.Count) content type(s)."
        
        foreach ($contentType in $contentTypes) {
            Write-Verbose "Processing content type: $($contentType.Name)"
            
            $script:ContentTypeCollection += [PSCustomObject]@{
                Name        = $contentType.Name
                ID          = $contentType.Id.StringValue
                Group       = $contentType.Group
                Description = $contentType.Description
                Scope       = $contentType.Scope
                StringId    = $contentType.StringId
                SchemaXml   = $contentType.SchemaXml
            }
        }
    }
    catch {
        Write-Warning "Error retrieving content types: $($_.Exception.Message)"
        throw
    }
}

end {
    if ($script:ContentTypeCollection.Count -gt 0) {
        Write-Verbose "Exporting $($script:ContentTypeCollection.Count) content type(s) to CSV..."
        
        $script:ContentTypeCollection | Export-Csv -Path $csvPath -NoTypeInformation
        
        Write-Host "" -ForegroundColor Cyan
        Write-Host "========================================" -ForegroundColor Cyan
        Write-Host "  Content Type Export Complete" -ForegroundColor Green
        Write-Host "========================================" -ForegroundColor Cyan
        Write-Host "Total content types exported: $($script:ContentTypeCollection.Count)" -ForegroundColor White
        Write-Host "CSV file location: $csvPath" -ForegroundColor White
        Write-Host "Transcript log: $transcriptPath" -ForegroundColor White
        Write-Host "========================================" -ForegroundColor Cyan
        Write-Host "" -ForegroundColor Cyan
    }
    else {
        Write-Warning "No content types found to export."
    }
    
    Stop-Transcript | Out-Null
}

# Usage examples:
# Basic usage:
# .\Export-ContentTypes.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing"

# Custom output directory:
# .\Export-ContentTypes.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -OutputPath "C:\Reports"

# With verbose logging:
# .\Export-ContentTypes.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -Verbose

# Pipeline usage with current directory:
# "https://contoso.sharepoint.com/sites/marketing" | .\Export-ContentTypes.ps1
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati (https://github.com/chandaniprajapati) |
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-content-type-details-to-csv" aria-hidden="true" />



# Reports to Excel where the specified Term is used 

## Summary

This sample script reports where a specific Managed Metadata Term is used across your SharePoint tenant. It searches through all site collections and subsites, looking for lists with managed metadata fields that reference the specified Term Set and Term GUIDs. Results are exported to CSV for easy analysis.

This script uses CLI for Microsoft 365 v11.4.0+ and follows modern PowerShell best practices including typed parameters, begin/process/end structure, WhatIf support, comprehensive error handling, and progress reporting.


![Example Screenshot](assets/ReportTermUse.png)

> [!Note]
> For this sample, you will require PnP.Powershell to be installed

# [PnP PowerShell](#tab/pnpps)

```powershell

#Set Variables
$rootSiteURL = "https://devenvironment.sharepoint.com//"
$targetTermSetId = "cca0fb68-25e6-4b83-a998-9ad4e82c68f8"
$targetTermId = "6816b2d7-ea71-4906-8f25-e45acc32ede2"
$outputPath = "C:\temp\whereisthatTermidused.csv" 
#Get Credentials to connect
if(-not $Cred)
{
    $Cred = Get-Credential
}

function LookForTermInWeb ($web)
{
    #check root first
    $lists = Get-PnPList -Web $web -Connection $connection
    foreach($list in $lists)
    {
        $taxfields = Get-PnPField -List $list -Connection $connection | Where-Object {$_.TypeAsString -eq "TaxonomyFieldType"}
        foreach($taxfield in $taxfields)
        {
            if($taxfield.TermSetId -eq $targetTermSetId)
            {
                
                $listitems = Get-PnPListItem -List $list -Connection $connection
                foreach($listitem in $listitems)
                {
                    #get the termid value
                    $field = $listitem[$taxfield.InternalName]
                    if($field)
                    {
                        $termguid = $field.TermGuid
                        if($termguid -and $termguid -eq $targetTermId)
                        {
                            
                            $element = "" | Select-Object SiteUrl, Title, ListTitle, fieldname, fieldvalue
                            $element.SiteUrl = $Site.Url
                            $element.Title = $Site.Title
                            $element.ListTitle = $list.Title
                            $element.fieldname = $taxfield.Title
                            $element.fieldvalue = $field.Label
                            $outputArray.Add($element) | Out-Null
                        }
                    }
                    
                }
            }
        }
    }
}



$outputArray = [System.Collections.ArrayList]@()
 
Try {
    #Connect to PnP Online
    $connection = Connect-PnPOnline -Url $rootSiteURL -Credentials $Cred -ReturnConnection
 
    #Get All Site collections 
    $SitesCollections = Get-PnPTenantSite -Connection $connection

    Disconnect-PnPOnline -Connection $connection
    $index = 0
    #Loop through each site collection
    ForEach($Site in $SitesCollections) 
    { 
        
        Write-host -ForegroundColor Green "$($Site.Url ) , number $index of $($SitesCollections.Count)"
        $index++
        Try 
        {
            #Connect to site collection
            $connection = Connect-PnPOnline -Url $Site.Url -Credentials $Cred -ReturnConnection
            LookForTermInWeb -Web (Get-PnpWeb -Connection $connection)
            
            
            $SubSites = Get-PnPSubWeb -Recurse -Connection $connection
            ForEach ($web in $SubSites)
            {
                Write-host "Web  : $($Web.URL)"
                LookForTermInWeb -web $web
            }
        }
        Catch {
            write-host -f Red "`tError:" $_.Exception.Message
        }
        finally
        {
            if($connection)
            {
                Disconnect-PnPOnline -Connection $connection
            }
        }
    }
}
Catch {
    write-host -f Red "Error:" $_.Exception.Message
}

$outputArray  | Export-Csv -Path $outputPath -Force -Encoding utf8BOM -Delimiter "|"

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage="GUID of the Term Set to search")]
    [ValidatePattern('^[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}$')]
    [string]$TermSetId,
    
    [Parameter(Mandatory, HelpMessage="GUID of the specific Term to find")]
    [ValidatePattern('^[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}$')]
    [string]$TermId,
    
    [Parameter(Mandatory, HelpMessage="Output CSV file path")]
    [ValidateNotNullOrEmpty()]
    [string]$OutputPath,
    
    [Parameter(HelpMessage="Include hidden lists in search")]
    [switch]$IncludeHiddenLists,
    
    [Parameter(HelpMessage="Directory for transcript logs")]
    [string]$TranscriptPath = (Get-Location).Path
)

begin {
    $logFile = Join-Path $TranscriptPath "ReportTermUse-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
    Start-Transcript -Path $logFile
    
    Write-Host "Ensuring login to Microsoft 365..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) { throw "Failed to login to Microsoft 365" }
    
    $outputDir = Split-Path -Path $OutputPath -Parent
    if ($outputDir -and -not (Test-Path $outputDir)) {
        throw "Output directory does not exist: $outputDir"
    }
    
    $script:Results = [System.Collections.ArrayList]@()
    $script:SitesProcessed = 0
    $script:SitesFailed = 0
    $script:TermsFound = 0
    
    function Find-TermInWeb {
        param(
            [Parameter(Mandatory)]
            [string]$WebUrl,
            
            [Parameter(Mandatory)]
            [string]$SiteTitle
        )
        
        Write-Verbose "Checking web: $WebUrl"
        
        try {
            if ($IncludeHiddenLists) {
                $lists = m365 spo list list --webUrl $WebUrl --output json | ConvertFrom-Json
            } else {
                $lists = m365 spo list list --webUrl $WebUrl --filter "Hidden eq false" --output json | ConvertFrom-Json
            }
            
            if ($LASTEXITCODE -ne 0) { throw "Failed to retrieve lists from $WebUrl" }
            
            $lists = @($lists)
            
            foreach ($list in $lists) {
                Write-Verbose "  Checking list: $($list.Title)"
                
                try {
                    $fields = m365 spo field list --webUrl $WebUrl --listTitle $list.Title --output json | ConvertFrom-Json
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to get fields for list: $($list.Title)"
                        continue
                    }
                    
                    $fields = @($fields)
                    $taxFields = $fields | Where-Object { $_.TypeAsString -eq "TaxonomyFieldType" -and $_.TermSetId -eq $TermSetId }
                    
                    foreach ($taxField in $taxFields) {
                        Write-Verbose "    Checking taxonomy field: $($taxField.Title)"
                        
                        try {
                            $items = m365 spo listitem list --listTitle $list.Title --webUrl $WebUrl --output json | ConvertFrom-Json
                            if ($LASTEXITCODE -ne 0) {
                                Write-Warning "Failed to get items for list: $($list.Title)"
                                continue
                            }
                            
                            $items = @($items)
                            
                            foreach ($item in $items) {
                                $fieldValue = $item."$($taxField.InternalName)"
                                if ($fieldValue -and $fieldValue.TermGuid -eq $TermId) {
                                    if ($PSCmdlet.ShouldProcess("$WebUrl - $($list.Title)", "Record term usage")) {
                                        $result = [PSCustomObject]@{
                                            SiteUrl    = $WebUrl
                                            SiteTitle  = $SiteTitle
                                            ListTitle  = $list.Title
                                            FieldName  = $taxField.Title
                                            FieldValue = $fieldValue.Label
                                        }
                                        $script:Results.Add($result) | Out-Null
                                        $script:TermsFound++
                                        Write-Host "    ✓ Found term in: $($list.Title) - $($taxField.Title)" -ForegroundColor Green
                                    }
                                }
                            }
                        } catch {
                            Write-Warning "Error processing items in list '$($list.Title)': $_"
                            continue
                        }
                    }
                } catch {
                    Write-Warning "Error processing list '$($list.Title)': $_"
                    continue
                }
            }
        } catch {
            Write-Warning "Error retrieving lists from '$WebUrl': $_"
            throw
        }
    }
}

process {
    Write-Host "`nRetrieving all site collections..." -ForegroundColor Cyan
    
    try {
        $sites = m365 spo site list --output json | ConvertFrom-Json
        if ($LASTEXITCODE -ne 0) { throw "Failed to retrieve site collections" }
        
        $sites = @($sites)
        Write-Host "Found $($sites.Count) site collections`n" -ForegroundColor Green
    } catch {
        Write-Error "Failed to retrieve sites: $_"
        throw
    }
    
    $index = 0
    foreach ($site in $sites) {
        $index++
        $percentComplete = [math]::Round(($index / $sites.Count) * 100)
        Write-Progress -Activity "Scanning sites for term usage" -Status "$index of $($sites.Count): $($site.Url)" -PercentComplete $percentComplete
        
        Write-Host "[$index/$($sites.Count)] Processing: $($site.Url)" -ForegroundColor Cyan
        
        try {
            Find-TermInWeb -WebUrl $site.Url -SiteTitle $site.Title
            
            try {
                $subwebs = m365 spo web list --url $site.Url --output json | ConvertFrom-Json
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve subsites for: $($site.Url)"
                } else {
                    $subwebs = @($subwebs)
                    foreach ($subweb in $subwebs) {
                        Write-Verbose "  Processing subweb: $($subweb.Url)"
                        Find-TermInWeb -WebUrl $subweb.Url -SiteTitle $site.Title
                    }
                }
            } catch {
                Write-Warning "Error processing subsites for '$($site.Url)': $_"
            }
            
            $script:SitesProcessed++
        } catch {
            $script:SitesFailed++
            Write-Warning "Error processing site $($site.Url): $_"
            continue
        }
    }
    
    Write-Progress -Activity "Scanning sites for term usage" -Completed
}

end {
    if ($script:Results.Count -gt 0) {
        $script:Results | Export-Csv -Path $OutputPath -Force -Encoding utf8BOM -Delimiter "|" -NoTypeInformation
        Write-Host "`n✓ Results exported to: $OutputPath" -ForegroundColor Green
    } else {
        Write-Host "`n⚠ No term usage found" -ForegroundColor Yellow
    }
    
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Term Usage Report Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Sites Processed:    $script:SitesProcessed" -ForegroundColor Green
    Write-Host "Sites Failed:       $script:SitesFailed" -ForegroundColor $(if ($script:SitesFailed -gt 0) { "Red" } else { "Green" })
    Write-Host "Terms Found:        $script:TermsFound" -ForegroundColor $(if ($script:TermsFound -gt 0) { "Green" } else { "Yellow" })
    Write-Host "========================================`n" -ForegroundColor Cyan
    
    Stop-Transcript
}

# Example 1: Basic usage
# .\Report-TermUse.ps1 -TermSetId "cca0fb68-25e6-4b83-a998-9ad4e82c68f8" -TermId "6816b2d7-ea71-4906-8f25-e45acc32ede2" -OutputPath "C:\temp\term-usage.csv"

# Example 2: Test with WhatIf (dry run)
# .\Report-TermUse.ps1 -TermSetId "cca0fb68-25e6-4b83-a998-9ad4e82c68f8" -TermId "6816b2d7-ea71-4906-8f25-e45acc32ede2" -OutputPath "C:\temp\term-usage.csv" -WhatIf

# Example 3: Include hidden lists
# .\Report-TermUse.ps1 -TermSetId "cca0fb68-25e6-4b83-a998-9ad4e82c68f8" -TermId "6816b2d7-ea71-4906-8f25-e45acc32ede2" -OutputPath "C:\temp\term-usage.csv" -IncludeHiddenLists

# Example 4: With verbose output and custom transcript location
# .\Report-TermUse.ps1 -TermSetId "cca0fb68-25e6-4b83-a998-9ad4e82c68f8" -TermId "6816b2d7-ea71-4906-8f25-e45acc32ede2" -OutputPath "C:\temp\term-usage.csv" -Verbose -TranscriptPath "C:\Logs"
```
***

## Contributors

| Author(s) |
|-----------|
| [Kasper Bo Larsen](https://github.com/kasperbolarsen)|
| [Adam Wójcik](https://github.com/Adam-it)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/report-term-use" aria-hidden="true" />

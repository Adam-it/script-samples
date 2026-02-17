

# Sample on how to locate the classic Search Result Web part and check the Remove Duplicates setting

## Summary

Locate all the pages where a classic Search Result Web Part is used, and if the Remove Duplicates setting is True (which is the default) log the location in the csv file. The duplicate algoritm is basically broken and will often trim records that are NOT duplicates. This sample is available in both PnP PowerShell and CLI for Microsoft 365 v11.4.0+.

## Implementation

- Open VS Code
- Create a new file
- Write a script as below,
- Change the variables to target to your environment
- Run the script.
 
## Screenshot of Output 

![Example Screenshot](assets/preview.png)

# [PnP PowerShell](#tab/pnpps)
```powershell


#Purpose: locate all pages that contains a OOTB search result web part ( for checking number of returned items + duplicat check)

function Handle-Pages ($pages, $url) 
{
    foreach($page in $pages)
    {
         $wps = Get-PnPWebPart -ServerRelativePageUrl  $page["FileRef"]
         foreach($wp in $wps)
         {
             try {
                 if($wp.WebPart.Properties.FieldValues.ContainsKey("ResultsPerPage"))
                 {
                     $dataProviderJSON = $wp.WebPart.Properties.FieldValues["DataProviderJSON"]
                     $vals = ConvertFrom-Json $dataProviderJSON
 
                     Write-Host $page["FileRef"] ", TrimDuplicates = " $vals.TrimDuplicates ", ResultsPerPage" $wp.WebPart.Properties.FieldValues["ResultsPerPage"]    
                    if($true -eq $vals.TrimDuplicates )
                    {
                        #add to output
                        $myobj = [PSCustomObject]@{
                            url = $url
                            page = $page["FileRef"]
                            
                        }
                        $hits.Add($myobj)
            
                    }
                 }
                 
             }
             catch 
             {
                write-host "Exception in Web Part data extraction: $($_.Exception)"    
             }
             
         }
                 
    }    
    
}


$hits = New-Object -TypeName "System.Collections.ArrayList"
#$cred = Get-Credential
$tenantUrl =  "https://[tenant]-admin.sharepoint.com"  
$tenantConn = Connect-PnPOnline -Url $tenantUrl -UseWebLogin -ReturnConnection

# get all classic site collections
$classicSiteCollections = Get-PnPTenantSite -Template "STS#0" -Connection $tenantConn
Disconnect-PnPOnline -Connection $tenantConn

$classicSiteCollections.Count
foreach($classicSiteCollection in $classicSiteCollections)
{
    $classicSiteCollection.Url
    Connect-PnPOnline -Url $classicSiteCollection.Url -UseWebLogin
    $pages = Get-PnPListItem -List "Site Pages" 
    Handle-Pages -pages $pages -url $classicSiteCollection.Url

    $webs = Get-PnPSubWebs -Recurse 
    foreach($web in $webs)
    {
         try
         {    
             $pages = Get-PnPListItem -List "Site Pages" -Web $web -ErrorAction Stop
             Handle-Pages -pages $pages $web.Url
             
        }
        catch
        {
             write-host $web.Url -ForegroundColor Red
        }
     }
}
$hits | Export-Csv -Path C:\temp\searchwebpartswithtrimming.csv -Encoding UTF8 -Delimiter "|" -Force -NoTypeInformation
   
   

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory, HelpMessage="SharePoint admin center URL")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')]
    [string]$AdminUrl,
    
    [Parameter(HelpMessage="Output path for CSV report")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath)) {
            throw "Invalid output path: $OutputPath"
        }
    }
    
    $script:ReportCollection = [System.Collections.ArrayList]::new()
    $script:Summary = @{
        SitesProcessed = 0
        WebsProcessed = 0
        PagesProcessed = 0
        WebPartsScanned = 0
        PagesWithTrimming = 0
        Failures = 0
    }
    
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = Join-Path $OutputPath "SearchWebPartScan_$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Starting classic Search Result Web Part scan..." -ForegroundColor Cyan
    Write-Host "Admin URL: $AdminUrl`n" -ForegroundColor Cyan
    
    Write-Verbose "Getting all classic site collections (Template: STS#0)..."
    $sitesJson = m365 spo site list --webTemplate "STS#0" --output json
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve classic site collections"
    }
    
    $script:ClassicSites = @($sitesJson | ConvertFrom-Json)
    Write-Host "Found $($script:ClassicSites.Count) classic site collection(s) to scan`n" -ForegroundColor Green
}

process {
    foreach ($site in $script:ClassicSites) {
        $siteUrl = $site.Url
        Write-Host "Processing site: $siteUrl" -ForegroundColor Yellow
        $script:Summary.SitesProcessed++
        
        try {
            Write-Verbose "Getting all webs in site collection..."
            $websJson = m365 spo web list --url $siteUrl --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to get webs for site: $siteUrl"
                $script:Summary.Failures++
                continue
            }
            
            $webs = @($websJson | ConvertFrom-Json)
            $webs = @($site) + $webs
            
            foreach ($web in $webs) {
                $webUrl = $web.Url
                Write-Verbose "Processing web: $webUrl"
                $script:Summary.WebsProcessed++
                
                try {
                    Write-Verbose "Getting pages from Site Pages library..."
                    $pagesJson = m365 spo listitem list --webUrl $webUrl --listTitle "Site Pages" --fields "FileRef" --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Verbose "Site Pages library not found or inaccessible in web: $webUrl"
                        continue
                    }
                    
                    $pages = @($pagesJson | ConvertFrom-Json)
                    Write-Verbose "Found $($pages.Count) page(s) in web: $webUrl"
                    
                    foreach ($page in $pages) {
                        $pageRef = $page.FileRef
                        Write-Verbose "  Checking page: $pageRef"
                        $script:Summary.PagesProcessed++
                        
                        try {
                            $pageName = Split-Path $pageRef -Leaf
                            
                            Write-Verbose "  Getting web parts on page..."
                            $controlsJson = m365 spo page control list --pageName $pageName --webUrl $webUrl --output json 2>&1
                            if ($LASTEXITCODE -ne 0) {
                                Write-Verbose "  Failed to get controls for page: $pageRef"
                                continue
                            }
                            
                            $controls = @($controlsJson | ConvertFrom-Json)
                            
                            foreach ($control in $controls) {
                                $script:Summary.WebPartsScanned++
                                
                                try {
                                    Write-Verbose "    Checking control ID: $($control.id)"
                                    $controlDetailJson = m365 spo page control get --id $control.id --pageName $pageName --webUrl $webUrl --output json 2>&1
                                    if ($LASTEXITCODE -ne 0) {
                                        Write-Verbose "    Failed to get control details"
                                        continue
                                    }
                                    
                                    $controlDetail = $controlDetailJson | ConvertFrom-Json
                                    
                                    if ($controlDetail.webPartData) {
                                        $webPartData = $controlDetail.webPartData | ConvertFrom-Json
                                        
                                        if ($webPartData.properties.PSObject.Properties.Name -contains 'ResultsPerPage') {
                                            Write-Verbose "    Found Search Result Web Part"
                                            
                                            if ($webPartData.properties.DataProviderJSON) {
                                                $dataProvider = $webPartData.properties.DataProviderJSON | ConvertFrom-Json
                                                $trimDuplicates = $dataProvider.TrimDuplicates
                                                $resultsPerPage = $webPartData.properties.ResultsPerPage
                                                
                                                Write-Host "  Found: $pageRef | TrimDuplicates: $trimDuplicates | ResultsPerPage: $resultsPerPage" -ForegroundColor Cyan
                                                
                                                if ($trimDuplicates -eq $true) {
                                                    Write-Host "    ⚠ TrimDuplicates is enabled (problematic)" -ForegroundColor Yellow
                                                    $script:Summary.PagesWithTrimming++
                                                    
                                                    $reportEntry = [PSCustomObject]@{
                                                        SiteUrl = $siteUrl
                                                        WebUrl = $webUrl
                                                        PageUrl = $pageRef
                                                        ResultsPerPage = $resultsPerPage
                                                        TrimDuplicates = $trimDuplicates
                                                    }
                                                    $null = $script:ReportCollection.Add($reportEntry)
                                                }
                                            }
                                        }
                                    }
                                }
                                catch {
                                    Write-Verbose "    Error processing control: $($_.Exception.Message)"
                                    continue
                                }
                            }
                        }
                        catch {
                            Write-Warning "  Error processing page $pageRef: $($_.Exception.Message)"
                            $script:Summary.Failures++
                            continue
                        }
                    }
                }
                catch {
                    Write-Warning "Error processing web $webUrl: $($_.Exception.Message)"
                    $script:Summary.Failures++
                    continue
                }
            }
        }
        catch {
            Write-Warning "Error processing site $siteUrl: $($_.Exception.Message)"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Write-Host "`n=== Scan Summary ===" -ForegroundColor Cyan
    Write-Host "Sites processed: $($script:Summary.SitesProcessed)" -ForegroundColor White
    Write-Host "Webs processed: $($script:Summary.WebsProcessed)" -ForegroundColor White
    Write-Host "Pages scanned: $($script:Summary.PagesProcessed)" -ForegroundColor White
    Write-Host "Web parts scanned: $($script:Summary.WebPartsScanned)" -ForegroundColor White
    Write-Host "Pages with TrimDuplicates enabled: $($script:Summary.PagesWithTrimming)" -ForegroundColor $(if ($script:Summary.PagesWithTrimming -gt 0) { 'Yellow' } else { 'Green' })
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures: 0" -ForegroundColor Green
    }
    
    if ($script:ReportCollection.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "SearchWebPartsWithTrimming_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "`nReport exported to: $csvPath" -ForegroundColor Green
    } else {
        Write-Host "`nNo pages with TrimDuplicates enabled found" -ForegroundColor Green
    }
    
    Stop-Transcript
}

# .\Find-SearchWebPartsWithTrimDuplicates.ps1 -AdminUrl "https://contoso-admin.sharepoint.com"

# .\Find-SearchWebPartsWithTrimDuplicates.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -Verbose

# .\Find-SearchWebPartsWithTrimDuplicates.ps1 -AdminUrl "https://contoso-admin.sharepoint.com" -OutputPath "C:\Reports"

# .\Find-SearchWebPartsWithTrimDuplicates.ps1 -AdminUrl "https://contoso.sharepoint.com"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Kasper Larsen, Fellowmind|
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-search-result-webparts" aria-hidden="true" />

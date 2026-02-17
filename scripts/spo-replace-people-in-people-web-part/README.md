

# Replace specific users in the People web part

## Summary

When people leave the company or assigned new responsibilities, you might want to replace them in the People web part with another user. This script will help you do that. Available in both PnP PowerShell and CLI for Microsoft 365 v11.4.0+.

![Example Screenshot](assets/example.png)


# [PnP PowerShell](#tab/pnpps)

```powershell

#define which site collections you wish to iterate
$tenentUrl = "https://contoso.sharepoint.com"
if(-not $conn)
{
    $conn = Connect-PnPOnline -Url $tenentUrl -Interactive -ReturnConnection
}

$relevantsitecollections = Get-PnPTenantSite -Connection $conn | Where-Object {$_.Url -eq "https://contoso.sharepoint.com/sites/HubsiteA"}

$SPAdminUrl = "https://contoso-admin.sharepoint.com/"
if(-not $SPAdminUrl)
{
    $spAdminConn = Connect-PnPOnline -Url $SPAdminUrl -Interactive -ReturnConnection
}


#replacementlist is a hashtable where the key is the old user and the value is the new user
$replacementlist = @{"i:0#.f|membership|pattif@tcwlv.onmicrosoft.com" = "i:0#.f|membership|adelev@tcwlv.onmicrosoft.com"}
    
$Output = @()

function UpdateWebPartIfRequired ($theWebpart, $page, $pageUrl)
{
    $props =  $thewebpart.PropertiesJson | ConvertFrom-Json
    $anyUpdates= $false
    foreach($person in $props.persons)
    {
        $personId = $person.Id
        if($replacementlist.ContainsKey($personId))
        {
            $anyUpdates = $true
            $newPersonId = $replacementlist[$personId]
            $person.Id = $newPersonId

            $myObject = [PSCustomObject]@{
            URL     = $tenentUrl+$page["FileRef"]
            errorcode = "User $personId has been replaced with $newPersonId"
            }        
            $Output+=($myObject)
            
        }
    }
    if($anyUpdates)
    {
        $thewebpart.PropertiesJson = $props | ConvertTo-Json
        $null = $page.Save()        
        $null = $page.Publish()
    }
    
}
foreach($site in  $relevantsitecollections)
{
    $sitecollectionUrl = $site.Url
    Write-Host "Url =  $sitecollectionUrl" -ForegroundColor Yellow
    
    $localConn = Connect-PnPOnline -Url $sitecollectionUrl -Interactive -ReturnConnection
    $pages = Get-PnPListItem -List "sitePages" -Connection $localConn

    foreach($page in $pages)
    {
        try 
        {
            $fullUrl = $tenentUrl+$page["FileRef"]
            Write-Host " Page = $fullUrl" -ForegroundColor Green
            $webpartpage = Get-PnPClientSidePage -Identity $page["FileLeafRef"] -ErrorAction Stop -Connection $localConn
            $webparts = $webpartpage.controls | Where-Object {$_.PropertiesJson -like "*persons*"}
            foreach($webpart in $webparts)
            {
                UpdateWebPartIfRequired -theWebpart $webpart -page $webpartpage -pageUrl $fullUrl
            }
        }
        catch 
        {
            $myObject = [PSCustomObject]@{
                URL     = $tenentUrl+$page["FileRef"]
                personid = ""
                personupn = ""
                errorcode = $_.Exception.Message

            }        
            $Output+=($myObject)
        }
    }
}
$Output | Export-Csv  -Path c:\temp\PeopleWebPartHasBeenUpdated.csv -Encoding utf8NoBOM -Force  -Delimiter "|"
  

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage="Array of SharePoint site URLs to process")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)')]
    [string[]]$SiteUrls,
    
    [Parameter(Mandatory, HelpMessage="Hashtable mapping old user IDs to new user IDs (e.g., @{'i:0#.f|membership|old@domain.com' = 'i:0#.f|membership|new@domain.com'})")]
    [hashtable]$ReplacementMap,
    
    [Parameter(HelpMessage="Path to save the CSV report")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }
    
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path $OutputPath)) {
            throw "OutputPath does not exist: $OutputPath"
        }
    }
    
    $script:ReportCollection = [System.Collections.ArrayList]::new()
    $script:Summary = @{
        SitesProcessed = 0
        PagesScanned = 0
        WebPartsUpdated = 0
        ReplacementsMade = 0
        Failures = 0
    }
    
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = Join-Path $OutputPath "ReplacePeopleWebPart_$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Starting People web part user replacement..." -ForegroundColor Cyan
    Write-Host "Sites to process: $($SiteUrls.Count)" -ForegroundColor Cyan
    Write-Host "User mappings: $($ReplacementMap.Count)" -ForegroundColor Cyan
    Write-Host ""
}

process {
    foreach ($siteUrl in $SiteUrls) {
        $script:Summary.SitesProcessed++
        Write-Host "Processing site: $siteUrl" -ForegroundColor Yellow
        
        try {
            Write-Verbose "Retrieving pages from Site Pages library..."
            $pagesJson = m365 spo listitem list --listTitle "Site Pages" --webUrl $siteUrl --fields "FileLeafRef,FileRef" --output json
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve pages from site: $siteUrl"
                $script:Summary.Failures++
                continue
            }
            
            $pages = @($pagesJson | ConvertFrom-Json)
            $aspxPages = $pages | Where-Object { $_.FileLeafRef -like "*.aspx" }
            
            Write-Host "  Found $($aspxPages.Count) pages to scan" -ForegroundColor Green
            
            foreach ($page in $aspxPages) {
                $script:Summary.PagesScanned++
                $pageUrl = $page.FileRef
                
                try {
                    Write-Verbose "  Processing page: $pageUrl"
                    
                    $controlsJson = m365 spo page control list --pageName $page.FileLeafRef --webUrl $siteUrl --output json
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "    Failed to get controls for page: $pageUrl"
                        $script:Summary.Failures++
                        continue
                    }
                    
                    $controls = @($controlsJson | ConvertFrom-Json)
                    
                    $peopleWebParts = $controls | Where-Object { 
                        $_.title -eq "People" -or ($_.webPartData -and $_.webPartData -like '*"persons"*')
                    }
                    
                    if ($peopleWebParts.Count -eq 0) {
                        Write-Verbose "    No People web parts found on page: $pageUrl"
                        continue
                    }
                    
                    Write-Verbose "    Found $($peopleWebParts.Count) People web part(s)"
                    
                    foreach ($control in $peopleWebParts) {
                        try {
                            Write-Verbose "      Processing web part ID: $($control.id)"
                            
                            $webPartJson = m365 spo page control get --id $control.id --pageName $page.FileLeafRef --webUrl $siteUrl --output json
                            if ($LASTEXITCODE -ne 0) {
                                Write-Warning "        Failed to get web part properties for ID: $($control.id)"
                                $script:Summary.Failures++
                                continue
                            }
                            
                            $webPartData = $webPartJson | ConvertFrom-Json
                            $webPartDataJson = $webPartData.webPartData | ConvertFrom-Json
                            
                            if (-not $webPartDataJson.properties -or -not $webPartDataJson.properties.persons) {
                                Write-Verbose "        Web part does not contain persons array"
                                continue
                            }
                            
                            $needsUpdate = $false
                            $replacementsInWebPart = 0
                            
                            foreach ($person in $webPartDataJson.properties.persons) {
                                $personId = $person.id
                                
                                if ($ReplacementMap.ContainsKey($personId)) {
                                    $newPersonId = $ReplacementMap[$personId]
                                    
                                    Write-Verbose "        Replacing user: $personId -> $newPersonId"
                                    $person.id = $newPersonId
                                    $needsUpdate = $true
                                    $replacementsInWebPart++
                                    $script:Summary.ReplacementsMade++
                                    
                                    $script:ReportCollection.Add([PSCustomObject]@{
                                        SiteUrl = $siteUrl
                                        PageUrl = $pageUrl
                                        WebPartId = $control.id
                                        OldUserId = $personId
                                        NewUserId = $newPersonId
                                        Status = "Success"
                                    }) | Out-Null
                                }
                            }
                            
                            if ($needsUpdate) {
                                $updatedWebPartData = $webPartDataJson | ConvertTo-Json -Depth 100 -Compress
                                
                                if ($PSCmdlet.ShouldProcess($pageUrl, "Replace $replacementsInWebPart user(s) in People web part")) {
                                    m365 spo page control set --id $control.id --pageName $page.FileLeafRef --webUrl $siteUrl --webPartData $updatedWebPartData
                                    
                                    if ($LASTEXITCODE -eq 0) {
                                        m365 spo page set --name $page.FileLeafRef --webUrl $siteUrl --publish
                                        
                                        if ($LASTEXITCODE -eq 0) {
                                            Write-Host "        Updated web part and published page" -ForegroundColor Green
                                            $script:Summary.WebPartsUpdated++
                                        } else {
                                            Write-Warning "        Failed to publish page: $pageUrl"
                                            $script:Summary.Failures++
                                        }
                                    } else {
                                        Write-Warning "        Failed to update web part ID: $($control.id)"
                                        $script:Summary.Failures++
                                    }
                                }
                            }
                        }
                        catch {
                            Write-Warning "        Error processing web part ID $($control.id): $($_.Exception.Message)"
                            $script:Summary.Failures++
                            continue
                        }
                    }
                }
                catch {
                    Write-Warning "    Error processing page $pageUrl: $($_.Exception.Message)"
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
    if ($script:ReportCollection.Count -gt 0) {
        $csvPath = Join-Path $OutputPath "PeopleWebPartReplacements_$(Get-Date -Format 'yyyyMMdd_HHmmss').csv"
        $script:ReportCollection | Export-Csv -Path $csvPath -NoTypeInformation
        Write-Host "\nReport exported to: $csvPath" -ForegroundColor Green
    } else {
        Write-Host "\nNo replacements were made." -ForegroundColor Yellow
    }
    
    Write-Host "\n=== Replacement Summary ===" -ForegroundColor Cyan
    Write-Host "Sites processed: $($script:Summary.SitesProcessed)" -ForegroundColor White
    Write-Host "Pages scanned: $($script:Summary.PagesScanned)" -ForegroundColor White
    Write-Host "Web parts updated: $($script:Summary.WebPartsUpdated)" -ForegroundColor White
    Write-Host "Total replacements made: $($script:Summary.ReplacementsMade)" -ForegroundColor White
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures: 0" -ForegroundColor Green
    }
    
    Stop-Transcript
}

# .\Replace-PeopleWebPartUsers.ps1 -SiteUrls @("https://contoso.sharepoint.com/sites/HubA") -ReplacementMap @{"i:0#.f|membership|old@contoso.com" = "i:0#.f|membership|new@contoso.com"}

# .\Replace-PeopleWebPartUsers.ps1 -SiteUrls @("https://contoso.sharepoint.com/sites/HubA", "https://contoso.sharepoint.com/sites/HubB") -ReplacementMap @{"i:0#.f|membership|old@contoso.com" = "i:0#.f|membership|new@contoso.com"} -WhatIf

# .\Replace-PeopleWebPartUsers.ps1 -SiteUrls @("https://contoso.sharepoint.com/sites/HubA") -ReplacementMap @{"i:0#.f|membership|old1@contoso.com" = "i:0#.f|membership|new1@contoso.com"; "i:0#.f|membership|old2@contoso.com" = "i:0#.f|membership|new2@contoso.com"} -Verbose

# .\Replace-PeopleWebPartUsers.ps1 -SiteUrls @("https://contoso.sharepoint.com/sites/HubA") -ReplacementMap @{"i:0#.f|membership|old@contoso.com" = "i:0#.f|membership|new@contoso.com"} -OutputPath "C:\\Reports"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-replace-people-in-people-web-part" aria-hidden="true" />

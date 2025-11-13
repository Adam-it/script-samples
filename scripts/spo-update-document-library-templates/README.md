

# Add document templates to the New dropdown in a document library

## Summary

It is pretty easy to add document templates to a document library in SharePoint Online by hand, but we often have to provide those document templates as part of a provisioning solution. This script shows how to add document templates to a document library using PnP PowerShell and CLI for Microsoft 365.
The most important property in this sample is `view.NewDocumentTemplates`. This property is a JSON string that contains the templates that are shown in the **New** dropdown in the document library.
The CLI version automates copying document templates into the library Forms folder and updates the view metadata using JSON payloads.

![Example Screenshot](assets/example.png)



# [PnP PowerShell](#tab/pnpps)

```powershell

function AddDocumentTemplateToLibrary {
    param (
        [Parameter(Mandatory=$true)]
        [string]$targetUrl,
        [Parameter(Mandatory=$true)]
        [string]$templateFileUrl,
        [Parameter(Mandatory=$true)]
        [string]$libraryName,
        [Parameter(Mandatory=$true)]
        [string]$targetContentType,
        [Parameter(Mandatory=$true)]
        [string] $templateId,
        [Parameter(Mandatory=$false)]
        [string] $baseNewTemplatesAsJson
    )
    
    #add a doucment template from another site to the library using the NewDocumentTemplates 

    #in this case I assume that you want to create the library if it does not exist
    $list = Get-PnPList -Identity $libraryName -Connection $targetConn -ErrorAction SilentlyContinue
    if($null -eq $list)
    {
        New-PnPList -Title $libraryName -Template DocumentLibrary -Connection $targetConn
        $list = Get-PnPList -Identity $libraryName -Connection $targetConn
    }
      

    #add the document template to the librarys Forms folder
    if($templateFileUrl)
    {
        $targettemplateFileUrl = $targetConn.Url + "/$libraryName/Forms/"+ $templateFileUrl.Split("/")[-1]

        #we can't be sure if the URL is using "sites" or "teams" so we need to check for both
        if($templateFileUrl.IndexOf("/sites/") -gt -1)
        {
            $relativetemplateFileUrl = $templateFileUrl.Substring($templateFileUrl.IndexOf("/sites/"))
        }
        else 
        {
            if($templateFileUrl.IndexOf("/teams/") -gt -1)
            {
                $relativetemplateFileUrl = $templateFileUrl.Substring($templateFileUrl.IndexOf("/teams/"))
            }   
            else 
            {
                throw "The template file URL does not contain /sites/ or /teams/"
            }
            
        }
        
        #check if the file already exists in the library
        $file = Get-PnPFile -Url $targettemplateFileUrl -Connection $targetConn -AsFileObject -ErrorAction SilentlyContinue
        if(-not $file)
        {
            $localeTemplateContainer = "$($targetConn.Url)/$libraryName/Forms"
            Copy-PnPFile -SourceUrl $relativetemplateFileUrl -TargetUrl $localeTemplateContainer -Connection $targetConn -Force -ErrorAction Stop
            #I have seen that the file is not always available immediately after the copy, so we need to retry a few times
            $retryindex = 0
            while($retryindex -lt 5 -and -not $file)
            {
                Start-Sleep -Seconds 5
                $file = Get-PnPFile -Url $targettemplateFileUrl -Connection $targetConn -AsFileObject -ErrorAction SilentlyContinue
                $retryindex++
            }
            
            if(-not $file)
            {
                throw "Failed to copy the template file to the library"                
            }
        }             
    }

    $view = Get-PnPView -List $list -Connection $targetConn | Where-Object { $_.DefaultView -eq $true }
    #if no value is provided for the baseNewTemplatesAsJson, then use the default value
    if($baseNewTemplatesAsJson -eq $null -or $baseNewTemplatesAsJson -eq "")
    {
        $existingTemplates = $view.NewDocumentTemplates
        #feel free to add or remove templates to the list
        $baseNewTemplatesAsJson = '[{"visible":true,"title":"Folder","isFolder":false,"iconProps":{"iconName":"folder16_svg","className":"newDocumentCommandIcon_4913fd57"},"templateId":"NewFolder","order":0},{"visible":true,"title":"Word document","isFolder":false,"iconProps":{"iconName":"docx16_svg","aria-label":"docx","className":"newDocumentCommandIcon_4913fd57"},"templateId":"NewDOC","order":1},{"visible":true,"title":"Excel workbook","isFolder":false,"iconProps":{"iconName":"xlsx16_svg","aria-label":"xlsx","className":"newDocumentCommandIcon_4913fd57"},"templateId":"NewXSL","order":2},{"visible":true,"title":"PowerPoint presentation","isFolder":false,"iconProps":{"iconName":"pptx16_svg","aria-label":"pptx","className":"newDocumentCommandIcon_4913fd57"},"templateId":"NewPPT","order":3},{"visible":true,"title":"OneNote notebook","isFolder":false,"iconProps":{"iconName":"onetoc16_svg","aria-label":"onetoc","className":"newDocumentCommandIcon_4913fd57"},"templateId":"NewONE","order":4},{"visible":true,"title":"Excel survey","isFolder":false,"iconProps":{"iconName":"xlsx16_svg","aria-label":"xlsx","className":"newDocumentCommandIcon_4913fd57"},"templateId":"NewXSLSurvey","order":5},{"visible":true,"title":"Forms for Excel","isFolder":false,"iconProps":{"iconName":"xlsx16_svg","aria-label":"xlsx","className":"newDocumentCommandIcon_4913fd57"},"templateId":"NewXSLForm","order":6},{"visible":true,"title":"Visio drawing","isFolder":false,"iconProps":{"iconName":"vsdx16_svg","aria-label":"vsdx","className":"newDocumentCommandIcon_4913fd57"},"templateId":"NewVSDX","order":7}[ReplaceToken]]'
    }
    #get the ID of List Conten type for Document
    $ct = Get-PnPContentType -List $list -Connection $targetConn | Where-Object { $_.Name -eq $targetContentType }
    if($null -eq $ct)
    {
        throw "Failed to find the content type $targetContentType in the list $libraryName" 
    }
    $replacer = ',{"contentTypeId":"'+ $($ct.Id.StringValue) +'","isUpload":true,"templateId":"'+ $templateId +'","title":"'+ $templateId+'","url":"'+$targettemplateFileUrl+'","visible":true}'
    
    if($existingTemplates -ne $null)
    {
        $index = $existingTemplates.LastIndexOf("]")
        $baseNewTemplatesAsJson = $existingTemplates.Substring(0, $index) + "[ReplaceToken]]"
        
    }
    $newtemplatesAsJson = $baseNewTemplatesAsJson -replace "\[ReplaceToken\]", $replacer
    
    $view.NewDocumentTemplates = $newtemplatesAsJson
    $view.Update()
    Invoke-PnPQuery -Connection $targetConn
    
}

$libraryName = "Docs2"
$targetUrl = "https://contoso.sharepoint.com/sites/site1"
$templateFileUrl = "https://contoso.sharepoint.com/sites/GlobalTemplateSite/SiteAssets/PnP Modern Search Q.docx"

if($targetConn -eq $null)
{
    $targetConn = Connect-PnPOnline -Url $targetUrl -Interactive -ReturnConnection -ClientId "your id"
}

if($sourceConn -eq $null)
{
    $sourceConn = Connect-PnPOnline -Url $templateFileUrl -Interactive -ReturnConnection -ClientId "your id"
}

AddDocumentTemplateToLibrary -targetUrl $targetUrl -templateFileUrl $templateFileUrl -templateId "PnP Modern Search Q" -libraryName $libraryName -targetContentType "Document" 

$templateFileUrl = "https://contoso.sharepoint.com/sites/GlobalTemplateSite/SiteAssets/Building%20Blocks.dotx"
AddDocumentTemplateToLibrary -targetUrl $targetUrl -templateFileUrl $templateFileUrl -templateId "Building Blocks" -libraryName $libraryName -targetContentType "Document" 

$templateFileUrl = "https://contoso.sharepoint.com/sites/somesite/Shared%20Documents/Fejl.docx"
AddDocumentTemplateToLibrary -targetUrl $targetUrl -templateFileUrl $templateFileUrl -templateId "Fejl" -libraryName $libraryName -targetContentType "Document" 




```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
# .\Update-LibraryNewTemplates.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/site1" -LibraryTitle "Docs2" -ContentTypeName "Document" -TemplateSourceSiteUrl "https://contoso.sharepoint.com/sites/templates" -TemplateFileUrls @("SiteAssets/Proposal.docx") -ReportPath ".\reports\template-run.json" -WhatIf
[CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'Medium')]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Target SharePoint site URL")]
    [ValidatePattern('^https://')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Title of the document library to update")]
    [ValidateNotNullOrEmpty()]
    [string]$LibraryTitle,

    [Parameter(Mandatory = $true, HelpMessage = "Content type name to associate with templates")]
    [ValidateNotNullOrEmpty()]
    [string]$ContentTypeName,

    [Parameter(Mandatory = $true, HelpMessage = "Source SharePoint site URL hosting the templates")]
    [ValidatePattern('^https://')]
    [string]$TemplateSourceSiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "Server-relative or site-relative template URLs within the source site")]
    [ValidateNotNullOrEmpty()]
    [string[]]$TemplateFileUrls,

    [Parameter(HelpMessage = "Optional path for JSON status report")]
    [string]$ReportPath
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 session"
    m365 login --ensure

    if ($ReportPath) {
        $reportDir = Split-Path -Path $ReportPath -Parent
        if ($reportDir -and -not (Test-Path $reportDir)) {
            New-Item -ItemType Directory -Path $reportDir -Force | Out-Null
        }
    }

    $Script:TemplateResults = [System.Collections.Generic.List[pscustomobject]]::new()
    $Script:TemplateSourceUri = [System.Uri]$TemplateSourceSiteUrl
}

process {
    Write-Verbose "Resolving library information"
    $listJson = m365 spo list get --webUrl $SiteUrl --title $LibraryTitle --query '{Id:Id, FormsFolder:RootFolder.ServerRelativeUrl, DefaultView:DefaultView}' --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to resolve library '$LibraryTitle'. CLI output: $listJson"
    }

    $listInfo = $listJson | ConvertFrom-Json
    $listId = $listInfo.Id
    $formsFolder = $listInfo.FormsFolder.TrimEnd('/') + '/Forms'

    Write-Verbose "Retrieving default list view"
    $viewJson = m365 spo list view list --webUrl $SiteUrl --listId $listId --query "[?DefaultView==\`true\`] | [0]" --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve default view. CLI output: $viewJson"
    }

    if ([string]::IsNullOrWhiteSpace(($viewJson | Out-String).Trim())) {
        throw "Default view not found for library '$LibraryTitle'."
    }

    try {
        $view = $viewJson | ConvertFrom-Json
    }
    catch {
        throw "Failed to parse default view response. $_"
    }

    $existingTemplates = @()
    if ($view.NewDocumentTemplates) {
        if ($view.NewDocumentTemplates -is [string]) {
            try {
                $existingTemplates = $view.NewDocumentTemplates | ConvertFrom-Json
            }
            catch {
                $existingTemplates = @()
            }
        }
        else {
            $existingTemplates = $view.NewDocumentTemplates
        }
    }
    if ($existingTemplates -eq $null) {
        $existingTemplates = @()
    }

    $existingTemplates = @($existingTemplates)

    Write-Verbose "Resolving content type"
    $contentTypeJson = m365 spo list contenttype list --webUrl $SiteUrl --listId $listId --query "[?Name=='$ContentTypeName'] | [0]" --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to resolve content type '$ContentTypeName'. CLI output: $contentTypeJson"
    }

    if ([string]::IsNullOrWhiteSpace(($contentTypeJson | Out-String).Trim())) {
        throw "Content type '$ContentTypeName' was not found in library '$LibraryTitle'."
    }

    try {
        $contentType = $contentTypeJson | ConvertFrom-Json
    }
    catch {
        throw "Failed to parse content type response. $_"
    }

    if (-not $contentType) {
        throw "Content type '$ContentTypeName' was not found in library '$LibraryTitle'."
    }

    foreach ($templateUrl in $TemplateFileUrls) {
        Write-Verbose "Processing template '$templateUrl'"

        try {
            $serverRelativeTemplateUrl = if ($templateUrl.StartsWith('http')) {
                $uri = [System.Uri]$templateUrl
                if ($uri.GetLeftPart([System.UriPartial]::Authority).TrimEnd('/') -ne $Script:TemplateSourceUri.GetLeftPart([System.UriPartial]::Authority).TrimEnd('/')) {
                    throw "Template host '$($uri.Host)' does not match source site '$($Script:TemplateSourceUri.Host)'"
                }
                $uri.AbsolutePath
            }
            elseif ($templateUrl.StartsWith('/')) {
                $templateUrl
            }
            else {
                $Script:TemplateSourceUri.AbsolutePath.TrimEnd('/') + '/' + $templateUrl.TrimStart('/')
            }
        }
        catch {
            $Script:TemplateResults.Add([pscustomobject]@{
                Template    = $templateUrl
                Destination = $null
                Status      = 'Failed'
                Details     = "Invalid template URL: $_"
            })
            continue
        }

        $fileName = Split-Path -Path $serverRelativeTemplateUrl -Leaf
        $destinationUrl = "$formsFolder/$fileName"

        $copyResult = m365 spo file copy --webUrl $TemplateSourceSiteUrl --sourceUrl $serverRelativeTemplateUrl --targetUrl $destinationUrl --nameConflictBehavior replace --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            $Script:TemplateResults.Add([pscustomobject]@{
                Template    = $templateUrl
                Destination = $destinationUrl
                Status      = 'Failed'
                Details     = $copyResult
            })
            continue
        }

        $templateName = [System.IO.Path]::GetFileNameWithoutExtension($fileName)
        $entry = [pscustomobject]@{
            contentTypeId = $contentType.StringId
            isUpload      = $true
            templateId    = $templateName
            title         = $templateName
            url           = $destinationUrl
            visible       = $true
        }

        $existingTemplates = @($existingTemplates | Where-Object { $_.templateId -ne $entry.templateId })
        $existingTemplates += $entry

        $Script:TemplateResults.Add([pscustomobject]@{
            Template    = $templateUrl
            Destination = $destinationUrl
            Status      = 'Copied'
            Details     = ''
        })
    }

    $payload = $existingTemplates | ConvertTo-Json -Depth 4

    if ($PSCmdlet.ShouldProcess($LibraryTitle, 'Update New document templates')) {
        $updateOutput = m365 spo list view set --webUrl $SiteUrl --listId $listId --id $view.Id --NewDocumentTemplates $payload 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to update view templates. CLI output: $updateOutput"
        }
    }
}

end {
    if ($ReportPath) {
        $Script:TemplateResults | ConvertTo-Json -Depth 4 | Out-File -FilePath $ReportPath -Encoding UTF8
        Write-Host "Report exported to '$ReportPath'" -ForegroundColor Green
    }

    $total = $Script:TemplateResults.Count
    $copied = ($Script:TemplateResults | Where-Object { $_.Status -eq 'Copied' }).Count
    $failed = ($Script:TemplateResults | Where-Object { $_.Status -eq 'Failed' })

    Write-Host "Templates processed: $total" -ForegroundColor Cyan
    Write-Host "Templates copied: $copied" -ForegroundColor Green
    if ($failed.Count -gt 0) {
        Write-Warning "Failed template operations detected"
        foreach ($item in $failed) {
            Write-Warning " - $($item.Template): $($item.Details)"
        }
    }
    else {
        Write-Host "No template copy failures detected." -ForegroundColor Green
    }
}
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-document-library-templates" aria-hidden="true" />

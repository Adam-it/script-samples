

# Download all the content type document templates files associated with a library

## Summary

The script will download all the document templates assigned to all content types in a library. I created this script as I needed to download the document templates assocaited to a library's content types and could not find as easy way to do it through the UI or did not want to download SharePoint Designer etc.

## Implementation

- Open Windows PowerShell ISE or VS Code
- Copy script below to your clipboard
- Paste script into your preferred editor
- Change config variables to reflect the site, library name & download location required


# [PnP PowerShell](#tab/pnpps)

```powershell

# SharePoint online site URL
$siteUrl = "https://contoso.sharepoint.com/sites/clientfacing"

# Display name of SharePoint document library
$libraryName = "Documents"

# Local path where document templates will be downloaded
$LocalPathForDownload = "c:\temp\"

# Connect to SharePoint online site
Connect-PnPOnline -Url $siteUrl -Interactive

# Get SharePoint list with content types
$list = Get-PnPList -Identity $libraryName -Includes ContentTypes

foreach ($CT in $list.ContentTypes | Where-Object {$_.ReadOnly -eq $false})
{
    if ($CT.DocumentTemplateUrl) {
        Write-Host "Downloading Document Template: $($CT.DocumentTemplate) for Content Type: $($CT.Name) to $LocalPathForDownload$($CT.DocumentTemplate)"

        # Download content type document template
        Get-PnPFile -Url $CT.DocumentTemplateUrl -Path $LocalPathForDownload -Filename $($CT.DocumentTemplate) -AsFile
    }
}

# Disconnect SharePoint online connection
Disconnect-PnPOnline

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Get-SpoContentTypeTemplates {
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter(Mandatory, HelpMessage = 'Absolute URL of the SharePoint site that hosts the document library.')]
        [ValidateNotNullOrEmpty()]
        [string]$SiteUrl,

        [Parameter(Mandatory, HelpMessage = 'Display name of the document library.')]
        [ValidateNotNullOrEmpty()]
        [string]$LibraryName,

        [Parameter(Mandatory, HelpMessage = 'Directory where the template files will be downloaded.')]
        [ValidateNotNullOrEmpty()]
        [string]$DownloadPath
    )

    begin {
        Write-Verbose 'Ensuring CLI for Microsoft 365 authentication'
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to sign in with CLI for Microsoft 365. CLI output: $loginOutput"
        }

        if (-not (Test-Path -LiteralPath $DownloadPath -PathType Container)) {
            Write-Verbose "Creating download directory '$DownloadPath'"
            New-Item -Path $DownloadPath -ItemType Directory -Force | Out-Null
        }

        $script:Summary = [ordered]@{
            Downloaded = 0
            Failed      = 0
            Errors      = New-Object System.Collections.Generic.List[string]
        }
    }

    process {
        Write-Verbose 'Retrieving content types for the target library'
        $contentTypesOutput = m365 spo list contenttype list --webUrl $SiteUrl --listTitle $LibraryName --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve content types. CLI output: $contentTypesOutput"
        }

        $contentTypes = $contentTypesOutput | ConvertFrom-Json -ErrorAction Stop

        foreach ($contentType in $contentTypes | Where-Object { $_.ReadOnly -eq $false -and $_.DocumentTemplateUrl }) {
            $templateFileName = $contentType.DocumentTemplate
            $templateUrl = $contentType.DocumentTemplateUrl
            $targetPath = Join-Path $DownloadPath $templateFileName

            if ($PSCmdlet.ShouldProcess($targetPath, "Download document template for content type '$($contentType.Name)'") ) {
                Write-Verbose "Downloading template '$templateFileName' from $templateUrl"
                $downloadOutput = m365 spo file get --webUrl $SiteUrl --url $templateUrl --asFile --path $targetPath 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to download template '$templateFileName'. CLI output: $downloadOutput"
                    $script:Summary.Failed++
                    $script:Summary.Errors.Add("$templateFileName: $downloadOutput")
                }
                else {
                    Write-Host "Downloaded template '$templateFileName' for content type '$($contentType.Name)' to $targetPath"
                    $script:Summary.Downloaded++
                }
            }
        }
    }

    end {
        Write-Host ''
        Write-Host 'Template download completed.'
        Write-Host "  Files downloaded : $($script:Summary.Downloaded)"
        Write-Host "  Files failed     : $($script:Summary.Failed)"

        if ($script:Summary.Errors.Count -gt 0) {
            Write-Warning 'Errors encountered during download:'
            foreach ($errorEntry in $script:Summary.Errors) {
                Write-Warning "  - $errorEntry"
            }
        }
    }
}

# Example usage
Get-SpoContentTypeTemplates -SiteUrl 'https://contoso.sharepoint.com/sites/clientfacing' -LibraryName 'Documents' -DownloadPath 'C:\temp' -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Leon Armston](https://github.com/LeonArmston) |
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/about) |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-list-download-contenttype-documenttemplate" aria-hidden="true" />

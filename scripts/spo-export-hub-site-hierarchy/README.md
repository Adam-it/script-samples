

# Export of the Hierarchy of Hub Sites

## Summary

[SharePoint allows to associate a hub site with another hub site](https://learn.microsoft.com/sharepoint/hub-to-hub-association). This script exports the SharePoint site hierarchy into a Markdown file to visualize the hub sites and its associated sites. It helps to understand the structure of SharePoint.

![Screenshot of the example](./assets/example.png)

The following is a sample of the output markdown:

```markdown
# SharePoint Sites in tenant [contoso]

## Hub Sites and Sites Associated with Hub Sites

Here are the hub sites and the sites associated with the hub sites. Hub sites are shown in bold.

- **[Hub Site A](https://contoso.sharepoint.com/sites/HubSiteA)**
  - [Site 1](https://contoso.sharepoint.com/sites/Site1)
  - [Site 2](https://contoso.sharepoint.com/sites/Site2)
  - **[Hub Site B](https://contoso.sharepoint.com/sites/HubSiteB)**
    - [Site 3](https://contoso.sharepoint.com/sites/Site3)
    - [Site 4](https://contoso.sharepoint.com/sites/Site4)
  - **[Hub Site C](https://contoso.sharepoint.com/sites/HubSiteC)**
    - [Site 5](https://contoso.sharepoint.com/sites/Site5)
    - [Site 6](https://contoso.sharepoint.com/sites/Site6)

- **[Hub Site D](https://contoso.sharepoint.com/sites/HubSiteA)**
  - [Site 7](https://contoso.sharepoint.com/sites/Site7)
  - [Site 8](https://contoso.sharepoint.com/sites/Site8)

## Sites that are not Hub Sites and are not Associated with any Hub Site

Here are the sites that are not hub sites and are not associated with any hub site.

- [Site 9](https://contoso.sharepoint.com/sites/Site9)
- [Site 10](https://contoso.sharepoint.com/sites/Site10)
```

The markdown file is created in the "SharePointHubSiteHierarchyReport" folder in MyDocuments.

![Screenshot of the execution screen](./assets/execution-screen.png)

> [!Note]
> - To run this script, it is required to be able to access the SharePoint Tenant Administration site.
> - Subsites are not exported.

# [PnP PowerShell](#tab/pnpps)

```powershell
# Target tenant name
$tenantName = '<Tenant Name>' # e.g. contoso

# Constants
$noHubSiteId = '00000000-0000-0000-0000-000000000000'
$adminUrl = "https://$tenantName-admin.sharepoint.com/"
$exportFolderName = 'SharePointHubSiteHierarchyReport'
$exportFolderPath = Join-Path ([Environment]::GetFolderPath('MyDocuments')) $exportFolderName
$timeStamp = (Get-Date).ToString('yyyyMMdd-HHmmss')
$markdownFilePath = Join-Path $exportFolderPath "$timeStamp-$tenantName.md"

# Function: Generate Markdown
function GenerateMarkdownForHubSite {
    param (
        $hubSite,
        $level,
        $hubSites,
        $tenantSites
    )

    $indent = "  " * ($level - 1)
    $hubSiteInfo = $tenantSites | Where-Object { $_.Url -eq $hubSite.SiteUrl }
    $hubTitle = $hubSiteInfo.Title ? $hubSiteInfo.Title : $hubSiteInfo.Url
    $markdown = "$indent- **[$hubTitle]($($hubSiteInfo.Url))**`r`n"

    $childSites = $tenantSites | Where-Object { $_.HubSiteId -eq $hubSite.SiteId -and $_.Url -ne $hubSite.SiteUrl }
    foreach ($childSite in $childSites) {
        $title = $childSite.Title ? $childSite.Title : $childSite.Url
        $markdown += "$indent  - [$title]($($childSite.Url))`r`n"
    }

    $childHubSites = $hubSites | Where-Object { $_.ParentHubSiteId -eq $hubSite.SiteId }
    foreach ($childHubSite in $childHubSites) {
        $markdown += (GenerateMarkdownForHubSite $childHubSite ($level + 1) $hubSites $tenantSites)
    }

    return $markdown
}

try {
    # Create the Data Export Folder
    if (-not (Test-Path $exportFolderPath -PathType Container)) {
        Write-Host "Creating Data Export Folder...Started" -ForegroundColor Yellow
        New-Item -Path $exportFolderPath -ItemType Directory -Force -ErrorAction Stop
        Write-Host "Creating Data Export Folder...Completed" -ForegroundColor Green
    }

    # Connect to SharePoint site
    Write-Host "Connecting to SharePoint site...Started" -ForegroundColor Yellow
    Connect-PnPOnline -Url $adminUrl -Interactive -ErrorAction Stop
    Write-Host "Connecting to SharePoint site...Completed" -ForegroundColor Green

    # Get tenant sites
    Write-Host "Retrieving tenant sites...Started" -ForegroundColor Yellow
    $tenantSites = Get-PnPTenantSite -ErrorAction Stop
    Write-Host "Retrieving tenant sites...Completed" -ForegroundColor Green

    # Get hub sites
    Write-Host "Retrieving hub sites...Started" -ForegroundColor Yellow
    $hubSites = Get-PnPHubSite -ErrorAction Stop
    Write-Host "Retrieving hub sites...Completed" -ForegroundColor Green

    # Generate Markdown
    Write-Host "Generating markdown...Started" -ForegroundColor Yellow
    $markdownText = @()
    $markdownText += "# SharePoint Sites in tenant [$($tenantName)]`r`n"

    # Hub Sites and Sites Associated with Hub Sites
    $markdownText += "## Hub Sites and Sites Associated with Hub Sites`r`n"
    $markdownText += "Here are the hub sites and the sites associated with the hub sites. Hub sites are shown in bold.`r`n"
    $parentHubSites = $hubSites | Where-Object { $_.ParentHubSiteId -eq $noHubSiteId }
    foreach ($parentHubSite in $parentHubSites) {
        $markdownText += (GenerateMarkdownForHubSite $parentHubSite 1 $hubSites $tenantSites)
    }

    # Hub Sites and Sites Associated with Hub Sites
    $markdownText += "## Sites that are not Hub Sites and are not Associated with any Hub Site`r`n"
    $markdownText += "Here are the sites that are not hub sites and are not associated with any hub site.`r`n"
    $standaloneSites = $tenantSites | Where-Object { $_.HubSiteId -eq $noHubSiteId }
    foreach ($standaloneSite in $standaloneSites) {
        $title = if ($standaloneSite.Title) { $standaloneSite.Title } else { $standaloneSite.Url }
        $markdownText += "- [$title]($($standaloneSite.Url))"
    }
    Write-Host "Generating markdown...Completed" -ForegroundColor Green

    # Save the Markdown file
    Write-Host "Saving markdown file...Started" -ForegroundColor Yellow
    $markdownText -join "`r`n" | Out-File -FilePath $markdownFilePath -Encoding UTF8 -ErrorAction Stop
    Write-Host "Saving markdown file...Completed" -ForegroundColor Green

    # Display successful file save message
    Write-Host "-".PadRight(50, "-")
    Write-Host "Markdown file is located at: $markdownFilePath"
    Write-Host "-".PadRight(50, "-")
}
catch {
    Write-Error "Error message: $($_.Exception.Message)"
}
finally {
    Disconnect-PnPOnline
}
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Export-SpoHubSiteHierarchy {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory, HelpMessage = 'Tenant name without the SharePoint domain suffix (for example, contoso).')]
        [ValidateNotNullOrEmpty()]
        [string]$TenantName,

        [Parameter(HelpMessage = 'Folder where the markdown report will be created.')]
        [ValidateNotNullOrEmpty()]
        [string]$OutputFolder = (Join-Path ([Environment]::GetFolderPath('MyDocuments')) 'SharePointHubSiteHierarchyReport')
    )

    begin {
        $noHubSiteId = '00000000-0000-0000-0000-000000000000'

        function New-HubSiteMarkdown {
            param (
                $hubSite,
                $level,
                $hubSites,
                $tenantSites
            )

            $indent = '  ' * [Math]::Max($level - 1, 0)
            $hubSiteInfo = $tenantSites | Where-Object { $_.Url -eq $hubSite.SiteUrl }

            $hubUrl = $hubSite.SiteUrl
            if (-not $hubUrl -and $hubSiteInfo) {
                $hubUrl = $hubSiteInfo.Url
            }

            $hubTitle = if ($hubSiteInfo -and $hubSiteInfo.Title) { $hubSiteInfo.Title } else { $hubUrl }
            if (-not $hubTitle) {
                $hubTitle = 'Unnamed Site'
            }

            $lines = [System.Collections.Generic.List[string]]::new()
            $lines.Add('{0}- **[{1}]({2})**' -f $indent, $hubTitle, $hubUrl)

            $childSites = $hubSite.AssociatedSites
            if ($childSites) {
                foreach ($childSite in $childSites) {
                    $childUrl = $childSite.SiteUrl
                    $childTitle = if ($childSite.Title) { $childSite.Title } else { $childUrl }
                    $lines.Add('{0}  - [{1}]({2})' -f $indent, $childTitle, $childUrl)
                }
            }

            $childHubSites = $hubSites | Where-Object { $_.ParentHubSiteId -eq $hubSite.SiteId }
            foreach ($childHubSite in $childHubSites) {
                $childLines = New-HubSiteMarkdown -hubSite $childHubSite -level ($level + 1) -hubSites $hubSites -tenantSites $tenantSites
                if ($childLines) {
                    [void]$lines.AddRange($childLines)
                }
            }

            return $lines
        }

        if (-not (Test-Path -LiteralPath $OutputFolder -PathType Container)) {
            Write-Verbose "Creating output folder '$OutputFolder'"
            New-Item -Path $OutputFolder -ItemType Directory -Force | Out-Null
        }

        Write-Verbose 'Ensuring CLI for Microsoft 365 authentication'
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to sign in with CLI for Microsoft 365. CLI output: $loginOutput"
        }

        $timeStamp = Get-Date -Format 'yyyyMMdd-HHmmss'
        $markdownFilePath = Join-Path $OutputFolder "$timeStamp-$TenantName.md"

        $script:Summary = [ordered]@{
            HubSites        = 0
            AssociatedSites = 0
            StandaloneSites = 0
            ReportPath      = $null
            Errors          = New-Object System.Collections.Generic.List[string]
        }
    }

    process {
        try {
            Write-Verbose 'Retrieving tenant sites from SharePoint Online'
            $tenantSitesOutput = m365 spo site list --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Errors.Add("Failed to retrieve tenant sites. CLI output: $tenantSitesOutput")
                throw "Failed to retrieve tenant sites. CLI output: $tenantSitesOutput"
            }
            $tenantSites = $tenantSitesOutput | ConvertFrom-Json -ErrorAction Stop

            Write-Verbose 'Retrieving hub sites (including associated sites)'
            $hubSitesOutput = m365 spo hubsite list --includeAssociatedSites --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Errors.Add("Failed to retrieve hub sites. CLI output: $hubSitesOutput")
                throw "Failed to retrieve hub sites. CLI output: $hubSitesOutput"
            }
            $hubSites = $hubSitesOutput | ConvertFrom-Json -ErrorAction Stop

            $script:Summary.HubSites = $hubSites.Count
            if ($hubSites) {
                $script:Summary.AssociatedSites = ($hubSites | ForEach-Object {
                        if ($_.AssociatedSites) { $_.AssociatedSites.Count } else { 0 }
                    } | Measure-Object -Sum).Sum
            }

            Write-Verbose 'Generating markdown report content'
            $markdownText = @()
            $markdownText += "# SharePoint Sites in tenant [$($TenantName)]`r`n"

            $markdownText += "## Hub Sites and Sites Associated with Hub Sites`r`n"
            $markdownText += "Here are the hub sites and the sites associated with the hub sites. Hub sites are shown in bold.`r`n"
            $parentHubSites = $hubSites | Where-Object { $_.ParentHubSiteId -eq $noHubSiteId }
            foreach ($parentHubSite in $parentHubSites) {
                $markdownText += (New-HubSiteMarkdown $parentHubSite 1 $hubSites $tenantSites)
            }

            $markdownText += "## Sites that are not Hub Sites and are not Associated with any Hub Site`r`n"
            $markdownText += "Here are the sites that are not hub sites and are not associated with any hub site.`r`n"
            $standaloneSites = $tenantSites | Where-Object { $_.HubSiteId -eq "/Guid(00000000-0000-0000-0000-000000000000)/" }
            $script:Summary.StandaloneSites = $standaloneSites.Count
            foreach ($standaloneSite in $standaloneSites) {
                $title = if ($standaloneSite.Title) { $standaloneSite.Title } else { $standaloneSite.Url }
                $markdownText += "- [$title]($($standaloneSite.Url))"
            }
            Write-Verbose 'Markdown content generated successfully'

            Write-Verbose "Saving markdown file to '$markdownFilePath'"
            $markdownText -join "`r`n" | Out-File -FilePath $markdownFilePath -Encoding UTF8 -ErrorAction Stop
            Write-Host "Markdown file is located at: $markdownFilePath"
            $script:Summary.ReportPath = $markdownFilePath
        }
        catch {
            Write-Error "Error message: $($_.Exception.Message)"
            if ($script:Summary -and $_.Exception.Message) {
                $script:Summary.Errors.Add($_.Exception.Message)
            }
        }
    }

    end {
        Write-Host ''
        Write-Host 'Hub site hierarchy export completed.'
        Write-Host "  Hub sites processed      : $($script:Summary.HubSites)"
        Write-Host "  Associated sites counted : $($script:Summary.AssociatedSites)"
        Write-Host "  Standalone sites counted : $($script:Summary.StandaloneSites)"

        if ($script:Summary.ReportPath) {
            Write-Host "  Report written to        : $($script:Summary.ReportPath)"
        }

        if ($script:Summary.Errors.Count -gt 0) {
            Write-Warning "Encountered $($script:Summary.Errors.Count) issue(s):"
            foreach ($error in $script:Summary.Errors) {
                Write-Warning "    $error"
            }
        }
    }
}

# Example
Export-SpoHubSiteHierarchy -TenantName 'contoso' -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s)        |
|------------------|
| [Tetsuya Kawahara](https://github.com/tecchan1107) |
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-hub-site-hierarchy" aria-hidden="true" />

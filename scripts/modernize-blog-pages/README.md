

# Modernize Blog Pages

## Summary

Converts all blog pages in a site, this includes:

- Conversion of blog pages
- Connecting to MFA or supplying credentials
- Includes Logging to File, log flushing into single log file

> [!note]
> This script uses the older [SharePoint PnP PowerShell Online module](https://www.powershellgallery.com/packages/SharePointPnPPowerShellOnline/3.29.2101.0)

![Example Screenshot](assets/modern-page.png)

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
function Convert-BlogPagesToModern {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory, HelpMessage = "URL of the classic blog site")]
        [ValidateNotNullOrEmpty()]
        [string]
        $SourceUrl,

        [Parameter(Mandatory, HelpMessage = "URL of the modern communication site target")]
        [ValidateNotNullOrEmpty()]
        [string]
        $TargetUrl,

        [Parameter(HelpMessage = "Directory where the conversion report and logs will be stored")]
        [ValidateNotNullOrEmpty()]
        [string]
        $OutputDirectory = "./reports",

        [Parameter(HelpMessage = "Switch to publish newly created modern pages as news")]
        [switch]
        $PublishModernPages,

        [Parameter(HelpMessage = "Switch to promote the first converted page as the site home page")]
        [switch]
        $SetFirstPageAsHome
    )

    begin {
        Write-Host "Ensuring Microsoft 365 CLI session..." -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to ensure Microsoft 365 CLI login. CLI output: $loginOutput"
        }

        $resolvedDirectory = Resolve-Path -Path $OutputDirectory -ErrorAction SilentlyContinue
        if (-not $resolvedDirectory) {
            Write-Verbose "Creating output directory '$OutputDirectory'."
            $null = New-Item -ItemType Directory -Path $OutputDirectory -Force
            $resolvedDirectory = Resolve-Path -Path $OutputDirectory
        }

        $timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
        $script:ReportPath = Join-Path -Path $resolvedDirectory.Path -ChildPath ("blog-modernization-$timestamp.csv")
        $script:Summary = [ordered]@{
            SourcePosts        = 0
            ModernPagesCreated = 0
            PromotedAsNews     = 0
            HomePageSet        = 0
            Failures           = 0
        }
        $script:Results = New-Object System.Collections.Generic.List[object]
        $script:CreatedPages = New-Object System.Collections.Generic.List[string]
    }

    process {
        Write-Host "Retrieving blog posts from $SourceUrl..." -ForegroundColor Yellow
        $postOutput = m365 spo listitem list --webUrl $SourceUrl --listTitle "Posts" --fields "ID,Title,FileRef,FileLeafRef,Created,Author/Title,Author/Name" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list blog posts. CLI output: $postOutput"
        }

        try {
            $posts = $postOutput | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse list items as JSON. $($_.Exception.Message)"
        }

        if (-not $posts) {
            Write-Warning "No blog posts found in the Posts list."
            return
        }

        $script:Summary.SourcePosts = $posts.Count
        Write-Host "Found $($posts.Count) blog post(s); converting to modern pages at $TargetUrl..." -ForegroundColor Cyan

        $pageIndex = 0
        foreach ($post in $posts) {
            $pageIndex++
            $targetPageName = "blog-post-$($post.Id).aspx"
            $targetPageUrl = "$TargetUrl/SitePages/$targetPageName"
            $publishFlag = $PublishModernPages.IsPresent
            $reportEntry = [ordered]@{
                SourceTitle       = $post.Title
                SourceUrl         = $post.FileRef
                ModernPageUrl     = $targetPageUrl
                Status            = ""
                Notes             = ""
            }

            if (-not $PSCmdlet.ShouldProcess($targetPageUrl, "Create modern page")) {
                $reportEntry.Status = "Skipped"
                $reportEntry.Notes = "WhatIf applied"
                $script:Results.Add([pscustomobject]$reportEntry)
                continue
            }

            Write-Host "Creating modern page for '$($post.Title)'..." -ForegroundColor Yellow
            $pageOutput = m365 spo page add --webUrl $TargetUrl --name $targetPageName --title $post.Title --layoutType Article 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Failures++
                $reportEntry.Status = "Failed"
                $reportEntry.Notes = "Page creation error: $pageOutput"
                $script:Results.Add([pscustomobject]$reportEntry)
                Write-Warning "Failed to create page for '$($post.Title)'. CLI output: $pageOutput"
                continue
            }

            $script:Summary.ModernPagesCreated++
            $script:CreatedPages.Add($targetPageUrl)

            if ($publishFlag) {
                Write-Host "Publishing modern page $targetPageName as news..." -ForegroundColor Cyan
                $publishOutput = m365 spo page publish --webUrl $TargetUrl --name $targetPageName --promoteAs NewsArticle 2>&1
                if ($LASTEXITCODE -eq 0) {
                    $script:Summary.PromotedAsNews++
                }
                else {
                    $script:Summary.Failures++
                    $reportEntry.Notes += " Publish failed: $publishOutput"
                    Write-Warning "Failed to publish page $targetPageName. CLI output: $publishOutput"
                }
            }

            $reportEntry.Status = "Created"
            $script:Results.Add([pscustomobject]$reportEntry)
        }

        if ($SetFirstPageAsHome -and $script:CreatedPages.Count -gt 0) {
            $homePageUrl = $script:CreatedPages[0]
            Write-Host "Setting $homePageUrl as the site home page..." -ForegroundColor Cyan
            $homeOutput = m365 spo web set --webUrl $TargetUrl --welcomePage ([System.IO.Path]::GetFileName($homePageUrl)) 2>&1
            if ($LASTEXITCODE -eq 0) {
                $script:Summary.HomePageSet++
            }
            else {
                $script:Summary.Failures++
                Write-Warning "Failed to set home page. CLI output: $homeOutput"
            }
        }
    }

    end {
        if ($script:Results.Count -gt 0) {
            try {
                $script:Results | Export-Csv -Path $script:ReportPath -NoTypeInformation -Encoding UTF8
                Write-Host "Modernization report saved to $($script:ReportPath)." -ForegroundColor Green
            }
            catch {
                $script:Summary.Failures++
                Write-Error "Failed to write CSV report. $($_.Exception.Message)"
            }
        }
        else {
            Write-Host "No pages were created during this run." -ForegroundColor Green
        }

        Write-Host "----- Summary -----" -ForegroundColor Cyan
        Write-Host "Blog posts processed      : $($script:Summary.SourcePosts)"
        Write-Host "Modern pages created      : $($script:Summary.ModernPagesCreated)"
        Write-Host "Pages promoted as news    : $($script:Summary.PromotedAsNews)"
        Write-Host "Home page set             : $($script:Summary.HomePageSet)"
        Write-Host "Failures encountered      : $($script:Summary.Failures)"
        if ($script:Results.Count -gt 0 -and (Test-Path -Path $script:ReportPath)) {
            Write-Host "Report path               : $($script:ReportPath)" -ForegroundColor Cyan
        }
    }
}

Convert-BlogPagesToModern -SourceUrl "https://contoso.sharepoint.com/sites/blog" -TargetUrl "https://contoso.sharepoint.com/sites/communicationsite" -PublishModernPages -SetFirstPageAsHome
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell

    # Classic blog site url
    $SourceUrl,

    # Target modern communication site url
    [string]$TargetUrl,

    # Supply credentials for multiple runs/sites
    $Credentials = Get-Credential

    # Specify log file location
    [string]$LogOutputFolder = "c:\temp"

    Connect-PnPOnline -Url $SourceUrl -Credentials $Credentials -Verbose
    Start-Sleep -s 3

    Write-Host "Modernizing blog pages..." -ForegroundColor Cyan

    $posts = Get-PnPListItem -List "Posts"

    Write-Host "pages fetched"

    Foreach($post in $posts)
    {
        $postTitle = $post.FieldValues["Title"]

        Write-Host " Processing blog post $($postTitle)"

        ConvertTo-PnPClientSidePage -Identity $postTitle `
                                    -BlogPage `
                                    -Overwrite `
                                    -TargetWebUrl $TargetUrl `
                                    -LogType File `
                                    -LogVerbose `
                                    -LogSkipFlush `
                                    -LogFolder $LogOutputFolder `
                                    -KeepPageCreationModificationInformation `
                                    -PostAsNews `
                                    -SetAuthorInPageHeader `
                                    -CopyPageMetadata
    }

    # Write the logs to the folder
    Save-PnPClientSidePageConversionLog

    Write-Host "Blog site modernization complete! :)" -ForegroundColor Green

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

## Contributors

| Author(s) |
|-----------|
| Bert Jansen |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]

```

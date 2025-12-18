

# Scan libraries for webhook and export to csv

## Summary

As part of the pre-migration assessment, it is important to identify all libraries that have webhooks configured. The CLI for Microsoft 365 version of this sample lets you target specific sites or filtered sets, collect library webhooks, and export results to CSV for quick review.

![Example Screenshot](assets/example.png)


# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Get-SpoLibrariesWithWebhooks {
    [CmdletBinding(SupportsShouldProcess)]
    param (
        [Parameter(Mandatory = $true, HelpMessage = "Root SharePoint URL, for example 'https://contoso.sharepoint.com'.")]
        [ValidatePattern('^https://[^/]+\.sharepoint\.com/?$')]
        [string]$TenantRootUrl,

        [Parameter(Mandatory = $false, HelpMessage = "Substring to match within site URLs (improves performance on large tenants).")]
        [string]$SiteUrlContains,

        [Parameter(Mandatory = $false, HelpMessage = "Explicit site URLs (absolute or server-relative); when supplied, site type filtering is ignored and only these sites are scanned.")]
        [string[]]$SiteUrls,

        [Parameter(Mandatory = $false, HelpMessage = "Limit enumeration to the selected site type when retrieving sites.")]
        [ValidateSet('All','TeamSite','CommunicationSite')]
        [string]$SiteType = 'All',

        [Parameter(Mandatory = $false, HelpMessage = "Destination CSV path (defaults to timestamped file in current directory).")]
        [string]$OutputPath
    )

    begin {
        Write-Verbose 'Ensuring CLI for Microsoft 365 authentication'
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to sign in with CLI for Microsoft 365. CLI output: $loginOutput"
        }

        $timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
        if (-not $OutputPath) {
            $OutputPath = Join-Path -Path (Get-Location) -ChildPath "libraries-with-webhooks_$timestamp.csv"
        }

        $directory = Split-Path -Path $OutputPath -Parent
        if ($directory -and -not (Test-Path -Path $directory -PathType Container)) {
            Write-Verbose "Creating export directory '$directory'"
            New-Item -ItemType Directory -Path $directory -Force | Out-Null
        }

        $script:Summary = [ordered]@{
            SitesScanned   = 0
            LibrariesScanned = 0
            WebhooksFound  = 0
            Records        = @()
            Failures       = @()
            ReportPath     = $OutputPath
        }

        $script:TenantRoot = $TenantRootUrl.TrimEnd('/')
        $script:SiteEnumerator = {
            param()

            if ($SiteUrls) {
                foreach ($rawUrl in $SiteUrls) {
                    if ([string]::IsNullOrWhiteSpace($rawUrl)) { continue }
                    $normalized = if ($rawUrl.StartsWith('http', 'InvariantCultureIgnoreCase')) {
                        $rawUrl.TrimEnd('/')
                    }
                    else {
                        "$($script:TenantRoot)/$($rawUrl.TrimStart('/'))"
                    }

                    Write-Verbose "Retrieving site information for $normalized"
                    $siteDetailsRaw = m365 spo site get --url $normalized --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        $script:Summary.Failures += [pscustomobject]@{
                            Stage   = 'GetSite'
                            Target  = $normalized
                            Message = $siteDetailsRaw
                        }
                        Write-Warning "Failed to retrieve site $normalized. CLI output: $siteDetailsRaw"
                        continue
                    }

                    $siteObj = @()
                    if (-not [string]::IsNullOrWhiteSpace($siteDetailsRaw)) {
                        $siteObj = @($siteDetailsRaw | ConvertFrom-Json)
                    }

                    if (-not $siteObj) {
                        continue
                    }

                    $siteRecord = [pscustomobject]@{
                        Url   = $siteObj[0].Url
                        Title = $siteObj[0].Title
                    }
                    $siteRecord
                }
                return
            }

            $filterParts = @("Url -like '$($script:TenantRoot.Replace("'","''"))%'")
            if ($SiteUrlContains) {
                $escapedMatch = $SiteUrlContains.Replace("'", "''")
                $filterParts += "Url -like '%$escapedMatch%'"
            }
            $filterExpression = [string]::Join(' and ', $filterParts)

            $siteArgs = @('spo','site','list','--output','json','--filter',$filterExpression)
            if ($SiteType -ne 'All') {
                $siteArgs += @('--type',$SiteType)
            }

            $queryValue = if ($SiteUrlContains) {
                $containsValue = $SiteUrlContains
                if ($containsValue -notmatch 'https?://') {
                    $containsValue = $containsValue.TrimStart('/')
                }
                $containsEscaped = $containsValue.Replace("'", "\'")
                "[?contains(Url, '$containsEscaped')]"
            }
            else {
                $tenantEscaped = $script:TenantRoot.Replace("'", "\'")
                "[?contains(Url, '$tenantEscaped')]"
            }
            $siteArgs += @('--query', $queryValue)

            Write-Verbose 'Retrieving SharePoint sites'
            $siteOutput = m365 @siteArgs 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to list SharePoint sites. CLI output: $siteOutput"
            }

            $sites = @()
            if (-not [string]::IsNullOrWhiteSpace($siteOutput)) {
                $sites = @($siteOutput | ConvertFrom-Json)
            }

            if (-not $sites) {
                Write-Warning 'No sites matched the specified filter.'
                return
            }

            foreach ($site in $sites) {
                [pscustomobject]@{
                    Url   = $site.Url
                    Title = $site.Title
                }
            }
        }
    }

    process {
        foreach ($siteRecord in & $script:SiteEnumerator) {
            $siteUrl = $siteRecord.Url
            $siteTitle = $siteRecord.Title
            $script:Summary.SitesScanned++
            Write-Verbose "Scanning libraries in $siteUrl"

            $listArgs = @(
                'spo','list','list',
                '--webUrl', $siteUrl,
                '--output','json',
                '--filter','BaseTemplate eq 101',
                '--properties','Id,Title,ParentWebUrl'
            )

            $listsOutput = m365 @listArgs 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Failures += [pscustomobject]@{
                    Stage   = 'ListLibraries'
                    Target  = $siteUrl
                    Message = $listsOutput
                }
                Write-Warning "Failed to enumerate libraries for $siteUrl. CLI output: $listsOutput"
                continue
            }

            $libraries = @()
            if (-not [string]::IsNullOrWhiteSpace($listsOutput)) {
                $libraries = @($listsOutput | ConvertFrom-Json)
            }

            if (-not $libraries) {
                continue
            }

            foreach ($library in $libraries) {
                $script:Summary.LibrariesScanned++
                $libraryId = $library.Id
                $libraryTitle = $library.Title

                $webhookOutput = m365 spo list webhook list --webUrl $siteUrl --listId $libraryId --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    $script:Summary.Failures += [pscustomobject]@{
                        Stage   = 'ListWebhooks'
                        Target  = "${siteUrl} | ${libraryTitle}"
                        Message = $webhookOutput
                    }
                    Write-Warning "Failed to retrieve webhooks for library '$libraryTitle' in site '$siteUrl'. CLI output: $webhookOutput"
                    continue
                }

                $webhooks = @()
                if (-not [string]::IsNullOrWhiteSpace($webhookOutput)) {
                    $webhooks = @($webhookOutput | ConvertFrom-Json)
                }

                if (-not $webhooks) {
                    continue
                }

                foreach ($webhook in $webhooks) {
                    $script:Summary.WebhooksFound++
                    $script:Summary.Records += [pscustomobject]@{
                        SiteUrl        = $siteUrl
                        SiteTitle      = $siteTitle
                        LibraryTitle   = $libraryTitle
                        LibraryId      = $libraryId
                        WebhookId      = $webhook.id
                        NotificationUrl = $webhook.notificationUrl
                        ExpirationDate  = $webhook.expirationDateTime
                        ClientState     = $webhook.clientState
                        ResourceId      = $webhook.resource
                    }
                }
            }
        }
    }

    end {
        if (-not $script:Summary.Records) {
            Write-Warning 'No libraries with webhooks were found.'
            return [pscustomobject]$script:Summary
        }

        if ($PSCmdlet.ShouldProcess($script:Summary.ReportPath, 'Export libraries with webhooks report')) {
            Write-Verbose "Exporting report to '$($script:Summary.ReportPath)'"
            $script:Summary.Records | Export-Csv -Path $script:Summary.ReportPath -NoTypeInformation -Encoding utf8 -Delimiter '|'
        }

        Write-Host '--- Webhook discovery summary ---'
        Write-Host "Sites scanned     : $($script:Summary.SitesScanned)"
        Write-Host "Libraries scanned : $($script:Summary.LibrariesScanned)"
        Write-Host "Webhooks found    : $($script:Summary.WebhooksFound)"
        Write-Host "Failures          : $($script:Summary.Failures.Count)"
        Write-Host "Report file       : $($script:Summary.ReportPath)"

        return [pscustomobject]$script:Summary
    }
}

Get-SpoLibrariesWithWebhooks -TenantRootUrl 'https://contoso.sharepoint.com' -SiteUrlContains "https://contoso.sharepoint.com/sites/" -Verbose

```

# [PnP PowerShell](#tab/pnpps)

```powershell


#use this function to get the sites you want to check
function GetSitesToCheck 
{
    #$allsites = Get-PnPTenantSite -Connection $adminConn -Filter "Url -like 'https://contoso.sharepoint.com/'"  -ErrorAction Stop
    $allsites = Get-PnPTenantSite -Connection $adminConn -ErrorAction Stop
    return $allsites
}


$adminUrl = "https://contoso-admin.sharepoint.com"
$PnPClientId = "Your PnP Client ID"
$adminConn = Connect-PnPOnline -Url $adminUrl -Interactive -ClientId $PnPClientId
$outputPath = "C:\temp\" 
$sites = GetSitesToCheck
$output = @()
foreach ($site in $sites) 
{
    $siteUrl = $site.Url
    $siteConn = Connect-PnPOnline -Url $siteUrl -Interactive -ClientId $PnPClientId
    $libraries = Get-PnPList  | Where-Object {$_.BaseType -eq "DocumentLibrary"}
    foreach ($library in $libraries) 
    {
        $webhooks = Get-PnPWebhookSubscription -List $library.Title
        foreach ($webhook in $webhooks)
        {
            Write-Host "Library $($library.Title) in site $($site.Title) has $($webhooks.Count) webhooks"
            $output += [PSCustomObject]@{
                SiteUrl = $siteUrl
                SiteTitle = $site.Title
                LibraryTitle = $library.Title
                WebhookId = $webhook.Id
                WebhookExpirationDateTime = $webhook.ExpirationDateTime
                WebhookNotificationUrl = $webhook.NotificationUrl
                WebhookResource = $webhook.Resource
                WebhookClientState = $webhook.ClientState
            }
        }
    }
}

$output | Export-Csv -Path "$outputPath\Webhooks.csv" -Encoding utf8BOM -Delimiter "|" -Force;


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-libraries-with-webhooks" aria-hidden="true" />

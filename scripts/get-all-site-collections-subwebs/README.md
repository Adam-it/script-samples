

# How to to get all site collections with their sub webs

## Summary

Sometimes we have a business requirement to get site collections with all the sub-webs so we can achieve the solution easily using PnP Powershell.

![Example Screenshot](assets/example.png)

result with CLI for Microsoft 365 version of the script

![Example Cli Screenshot](assets/example_cli.png)

Let's see step-by-step implementation

## Implementation

Open Windows Powershell ISE
Create a new file and write a script

Now we will see all the steps which we required to achieve the solution:

1. We will initialize the admin site URL, username, and password in the global variables.
2. Then we will create a Login function to connect the O365 SharePoint Admin site.
3. Create a function to get all site collections and all the sub-webs

So in the end, our script will be like this

# [PnP PowerShell](#tab/pnpps)

```powershell

$SiteURL = "https://domain-admin.sharepoint.com/"
$UserName = "UserName@domain.onmicrosoft.com"
$Password = "********"
$SecureStringPwd = $Password | ConvertTo-SecureString -AsPlainText -Force 
$Creds = New-Object System.Management.Automation.PSCredential -ArgumentList $UserName, $SecureStringPwd

Function Login {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $Creds)
    Write-Host "Connecting to Tenant Admin Site '$($SiteURL)'" 
    Connect-PnPOnline -Url $SiteURL -Credentials $creds
    Write-Host "Connection Successfull"
}

Function AllSiteCollAndSubWebs() {
    Login($Creds)
    $TenantSites = (Get-PnPTenantSite) | Select Title, Url       
       
    ForEach ( $TenantSite in $TenantSites) { 
        Connect-PnPOnline -Url $TenantSite.Url -Credentials $Creds
        Write-Host $TenantSite.Title $TenantSite.Url
        $subwebs = Get-PnPSubWebs -Recurse | Select Title, Url
        foreach ($subweb in $subwebs) { 
            Connect-PNPonline -Url $subweb.Url -Credentials $Creds
            Write-Host $subweb.Title $subweb.Url 
        }  
    }
}

AllSiteCollAndSubWebs

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
function Get-SpoSiteCollectionsWithSubwebs {
    [CmdletBinding()]
    param(
        [Parameter(HelpMessage = "Filter to apply when retrieving sites, e.g. `\"Url -like 'project'\"`")]
        [string] $Filter,

        [Parameter(HelpMessage = "Modern site types to include in the inventory")]
        [ValidateNotNullOrEmpty()]
        [ValidateSet('TeamSite', 'CommunicationSite')]
        [string[]] $SiteTypes = @('TeamSite', 'CommunicationSite'),

        [Parameter(HelpMessage = "Include classic site collections in the report")]
        [switch] $IncludeClassicSites,

        [Parameter(HelpMessage = "Include deleted sites in the report (sub-web enumeration is skipped)")]
        [switch] $IncludeDeletedSites,

        [Parameter(HelpMessage = "Optional path to export the results to CSV")]
        [string] $OutputPath,

        [Parameter(HelpMessage = "Emit the inventory objects to the pipeline")]
        [switch] $PassThru
    )

    begin {
        Write-Verbose "Ensuring CLI authentication"
        m365 login --ensure | Out-Null

        $results = New-Object System.Collections.Generic.List[psobject]
        $summary = [ordered]@{
            Categories = New-Object System.Collections.Generic.List[psobject]
            Errors     = 0
        }

        if ($OutputPath) {
            $directory = Split-Path -Path $OutputPath -Parent
            if (-not $directory) {
                $directory = '.'
            }
            if (-not (Test-Path -Path $directory)) {
                Write-Verbose "Creating directory '$directory'"
                New-Item -ItemType Directory -Path $directory -Force | Out-Null
            }
        }

        function Invoke-SiteCategory {
            param(
                [string] $Label,
                [string[]] $CommandArgs,
                [bool] $EnumerateSubwebs = $true
            )

            Write-Verbose "Retrieving $Label"
            $json = m365 @CommandArgs 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve $Label. CLI output: $json"
                $summary.Errors++
                return
            }

            if ([string]::IsNullOrWhiteSpace($json)) {
                Write-Verbose "No results returned for $Label"
                return
            }

            try {
                $sites = $json | ConvertFrom-Json
            }
            catch {
                Write-Warning "Failed to parse response for $Label. Error: $($_.Exception.Message)"
                $summary.Errors++
                return
            }

            $sites = @($sites)
            if ($sites.Count -eq 0) {
                Write-Verbose "No sites found for $Label"
                return
            }

            $stats = [ordered]@{
                Category = $Label
                Sites    = $sites.Count
                SubWebs  = 0
            }

            foreach ($site in $sites) {
                if ($EnumerateSubwebs) {
                    $webArgs = @(
                        'spo', 'web', 'list',
                        '--webUrl', $site.Url,
                        '--output', 'json',
                        '--query', '[].{Title:Title,Url:Url}'
                    )

                    $webJson = m365 @webArgs 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to retrieve sub-webs for '$($site.Url)'. CLI output: $webJson"
                        $summary.Errors++
                        $results.Add([pscustomobject]@{
                            Category     = $Label
                            SiteTitle    = $site.Title
                            SiteUrl      = $site.Url
                            SubWebTitle  = $null
                            SubWebUrl    = $null
                        }) | Out-Null
                        continue
                    }

                    try {
                        $subWebs = [string]::IsNullOrWhiteSpace($webJson) ? @() : @($webJson | ConvertFrom-Json)
                    }
                    catch {
                        Write-Warning "Failed to parse sub-webs for '$($site.Url)'. Error: $($_.Exception.Message)"
                        $summary.Errors++
                        $subWebs = @()
                    }

                    if ($subWebs.Count -eq 0) {
                        $results.Add([pscustomobject]@{
                            Category     = $Label
                            SiteTitle    = $site.Title
                            SiteUrl      = $site.Url
                            SubWebTitle  = $null
                            SubWebUrl    = $null
                        }) | Out-Null
                    }
                    else {
                        foreach ($subWeb in $subWebs) {
                            $results.Add([pscustomobject]@{
                                Category     = $Label
                                SiteTitle    = $site.Title
                                SiteUrl      = $site.Url
                                SubWebTitle  = $subWeb.Title
                                SubWebUrl    = $subWeb.Url
                            }) | Out-Null
                        }
                    }

                    $stats.SubWebs += $subWebs.Count
                }
                else {
                    $results.Add([pscustomobject]@{
                        Category     = $Label
                        SiteTitle    = $site.Title
                        SiteUrl      = $site.Url
                        SubWebTitle  = $null
                        SubWebUrl    = $null
                    }) | Out-Null
                }
            }

            $summary.Categories.Add([pscustomobject]$stats) | Out-Null
        }
    }

    process {
        foreach ($type in ($SiteTypes | Select-Object -Unique)) {
            $label = switch ($type) {
                'TeamSite'          { 'Team Sites' }
                'CommunicationSite' { 'Communication Sites' }
                Default             { $type }
            }

            $siteArgs = @(
                'spo', 'site', 'list',
                '--type', $type,
                '--output', 'json',
                '--query', '[].{Title:Title,Url:Url}'
            )

            if ($Filter) {
                $siteArgs += @('--filter', $Filter)
            }

            Invoke-SiteCategory -Label $label -CommandArgs $siteArgs
        }

        if ($IncludeClassicSites) {
            $classicArgs = @('spo', 'site', 'classic', 'list', '--output', 'json', '--query', '[].{Title:Title,Url:Url}')
            if ($Filter) {
                $classicArgs += @('--filter', $Filter)
            }
            Invoke-SiteCategory -Label 'Classic Sites' -CommandArgs $classicArgs
        }

        if ($IncludeDeletedSites) {
            $deletedArgs = @('spo', 'site', 'list', '--deleted', '--output', 'json', '--query', '[].{Title:Title,Url:Url}')
            if ($Filter) {
                $deletedArgs += @('--filter', $Filter)
            }
            Invoke-SiteCategory -Label 'Deleted Sites' -CommandArgs $deletedArgs -EnumerateSubwebs:$false
        }
    }

    end {
        if ($OutputPath) {
            Write-Verbose "Exporting results to '$OutputPath'"
            $results | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
            Write-Host "Results exported to $OutputPath" -ForegroundColor Green
        }

        Write-Host "Site inventory summary:" -ForegroundColor Cyan
        foreach ($stat in $summary.Categories) {
            Write-Host ("- {0}: {1} site(s), {2} sub web(s)" -f $stat.Category, $stat.Sites, $stat.SubWebs)
        }
        Write-Host ("- Errors: {0}" -f $summary.Errors)

        if ($PassThru) {
            $results
        }
    }
}

# Example usage
Get-SpoSiteCollectionsWithSubwebs -IncludeClassicSites -IncludeDeletedSites -OutputPath "./site-inventory.csv" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Source Credit

Sample first appeared on [How to to get all site collections with their sub webs using PnP PowerShell? | Microsoft 365 PnP Blog](https://techcommunity.microsoft.com/t5/microsoft-365-pnp-blog/how-to-to-get-all-site-collections-with-their-sub-webs-using-pnp/ba-p/2322131)

## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/get-all-site-collections-subwebs" aria-hidden="true" />

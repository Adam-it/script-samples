

# Associate Multiple Site Collections to Hub Site

## Summary

This PowerShell script can be used to associate mutilple site collections to Hub site. You can provide list of site collection URLs in an array.

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Update the parameters with site collection URLs and hub site URL
- Save the file and run it
 
# [PnP PowerShell](#tab/pnpps)

```powershell

# Parameters

# Provide SharePoint online Hub site URL
$HubSiteURL = "https://******.sharepoint.com/sites/**********"

# Array of site collections to associate with hub site
$arrSCs = @("https://******.sharepoint.com/sites/**********", "https://******.sharepoint.com/sites/**********", "https://******.sharepoint.com/sites/**********")

# Get admin user credentials
$creds = (Get-Credential)

function AssociateHubSite {
	try {
		foreach ($SC in $arrSCs) { 
			Write-Host "Connecting site collection: " $SC 
			Connect-PnPOnline -Url $SC -Credentials $creds
			Add-PnPHubSiteAssociation -Site $SC -HubSite $HubSiteURL -ErrorAction Stop
			Write-Host "Hub site associated with site collection: " $SC -ForegroundColor Green            
		}
	}
	catch {
		Write-Host "Error in associating hub site $($SC): " $_.Exception.Message -ForegroundColor Red
	}   
	
	# Disconnect SharePoint online connection
	Disconnect-PnPOnline
}

AssociateHubSite

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [SPO Management Shell](#tab/spoms-ps)

```powershell

# SharePoint tenant admin site collection URL
$adminSiteUrl = "https://contoso-admin.sharepoint.com"

# SharePoint online Hub site URL
$hubSiteURL = "https://contoso.sharepoint.com/sites/communicationhubsite"

# Array of site collections to associate with hub site
$arrayOfSites = @("https://contoso.sharepoint.com/sites/siteA", "https://contoso.sharepoint.com/sites/siteB", "https://contoso.sharepoint.com/sites/siteC")

# Connect to SharePoint Online admin site  
Connect-SPOService -Url $adminSiteUrl

function AssociateHubSite {
	try {
		foreach ($site in $arrayOfSites) { 
			Write-Host "Associating site collection: " $site 
			
			# Associating site collection with hub site
			Add-SPOHubSiteAssociation -Site $site -HubSite $hubSiteURL -ErrorAction Stop
			
			Write-Host "Hub site associated with site collection: " $site -ForegroundColor Green            
		}
	}
	catch {
		Write-Host "Error in associating hub site $($site): " $_.Exception.Message -ForegroundColor Red
	}   
	
	# Disconnect SharePoint online connection
	Disconnect-SPOService
}

AssociateHubSite

```

[!INCLUDE [More about SPO Management Shell](../../docfx/includes/MORE-SPOMS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Connect-SitesToHubWithCli {
    <#
    .SYNOPSIS
    Associates multiple SharePoint sites to a hub using CLI for Microsoft 365.

    .DESCRIPTION
    Ensures CLI authentication, resolves the hub site, iterates through the provided site URLs, and connects
    each site to the specified hub. Supports ShouldProcess for safe runs (`-WhatIf`).

    .PARAMETER HubSiteUrl
    URL of the hub site to which the sites should be connected.

    .PARAMETER SiteUrls
    Collection of site URLs to associate with the hub.

    .EXAMPLE
    Connect-SitesToHubWithCli -HubSiteUrl https://contoso.sharepoint.com/sites/communicationhubsite `
        -SiteUrls 'https://contoso.sharepoint.com/sites/siteA','https://contoso.sharepoint.com/sites/siteB'
    #>
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter(Mandatory, HelpMessage = 'URL of the target hub site.')]
        [ValidateNotNullOrEmpty()]
        [string]$HubSiteUrl,

        [Parameter(Mandatory, HelpMessage = 'One or more site URLs to connect to the hub.')]
        [ValidateNotNullOrEmpty()]
        [string[]]$SiteUrls
    )

    begin {
        Write-Host 'Ensuring CLI authentication.' -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "CLI login failed. Output: $loginOutput"
        }

        Write-Host "Resolving hub site '$HubSiteUrl'." -ForegroundColor Cyan
        $hubJson = m365 spo hubsite get --url $HubSiteUrl --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Unable to retrieve hub site details. CLI output: $hubJson"
        }

        try {
            $script:Hub = $hubJson | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse hub site information. $($_.Exception.Message)"
        }

        if (-not $script:Hub.Id) {
            throw 'Hub site ID could not be determined.'
        }
    }

    process {
        $script:Summary = [ordered]@{
            Processed = 0
            Connected = 0
            Skipped   = 0
            Failed    = 0
        }

        foreach ($siteUrl in $SiteUrls) {
            if ([string]::IsNullOrWhiteSpace($siteUrl)) {
                continue
            }

            Write-Verbose "Validating site '$siteUrl'."
            $siteJson = m365 spo site get --url $siteUrl --output json 2>&1
            if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($siteJson)) {
                Write-Warning "Site '$siteUrl' could not be retrieved. CLI output: $siteJson"
                $script:Summary.Failed++
                continue
            }

            try {
                $siteInfo = $siteJson | ConvertFrom-Json -ErrorAction Stop
            }
            catch {
                Write-Warning "Unable to parse site information for '$siteUrl'. $($_.Exception.Message)"
                $script:Summary.Failed++
                continue
            }

            $script:Summary.Processed++

            if ($siteInfo.hubSiteId -and $siteInfo.hubSiteId -eq $script:Hub.Id) {
                Write-Host "Site '$siteUrl' is already associated with the target hub; skipping." -ForegroundColor Yellow
                $script:Summary.Skipped++
                continue
            }

            if ($siteInfo.hubSiteId -and $siteInfo.hubSiteId -ne [Guid]::Empty -and $siteInfo.hubSiteId -ne $script:Hub.Id) {
                Write-Warning "Site '$siteUrl' is connected to a different hub (ID: $($siteInfo.hubSiteId))."
                $script:Summary.Skipped++
                continue
            }

            $action = "Connect site '$siteUrl' to hub '$HubSiteUrl'"
            if (-not $PSCmdlet.ShouldProcess($siteUrl, $action)) {
                $script:Summary.Skipped++
                continue
            }

            Write-Host $action -ForegroundColor Cyan
            $output = m365 spo site hubsite connect --siteUrl $siteUrl --id $script:Hub.Id 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to connect '$siteUrl'. CLI output: $output"
                $script:Summary.Failed++
            }
            else {
                Write-Host "Hub site associated with '$siteUrl'." -ForegroundColor Green
                $script:Summary.Connected++
            }
        }
    }

    end {
        Write-Host 'Hub association process completed.' -ForegroundColor Cyan
        Write-Host "Sites processed: $($script:Summary.Processed)" -ForegroundColor Cyan
        Write-Host "Sites connected: $($script:Summary.Connected)" -ForegroundColor Green
        Write-Host "Sites skipped: $($script:Summary.Skipped)" -ForegroundColor Yellow
        Write-Host "Sites failed: $($script:Summary.Failed)" -ForegroundColor Red
    }
}

# example usage
Connect-SitesToHubWithCli -HubSiteUrl 'https://contoso.sharepoint.com/sites/communicationhubsite' `
    -SiteUrls 'https://contoso.sharepoint.com/sites/siteA','https://contoso.sharepoint.com/sites/siteB'
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Siddharth Vaghasia](https://github.com/siddharth-vaghasia) |
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/about) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]

<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-associate-multiple-sites-to-hub" aria-hidden="true" />

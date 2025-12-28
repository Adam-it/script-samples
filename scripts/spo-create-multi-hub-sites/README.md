

# Create a multi-hub set of communication sites

## Summary

Want to see how the hub site association works but don't have an large intranet to play with, or looking to build a large multi-departmental intranet.
This sample script (available in both PnP PowerShell and CLI for Microsoft 365) builds 9 SharePoint communication sites, creates three hub sites and associates them to a main hub site; these are empty sites with no content, however you can build on this script or the approach to populate all the sites with example content.

> [!div class="full-image-size"]
>![Example Screenshot](assets/example.png)


# [PnP PowerShell](#tab/pnpps)

```powershell

[CmdletBinding()]
param (
    [string]$TenantOrg = "contoso",
    [int]$TimeZoneId = 2,
    [string]$OwnerEmail = "paul.bullock@contoso.onmicrosoft.com", # <email address>
    [string]$SiteListJsonFile = "large-intranet-example.json", #use the sample from the JSON tab
    $SiteType = "CommunicationSite" # <TeamSite|TeamSiteWithoutMicrosoft365Group|CommunicationSite>
)
begin {
   
    Start-Transcript -OutputDirectory .

    # Part 
    $adminUrl = "https://$($TenantOrg)-admin.sharepoint.com"

    # Requires SharePoint Admin Role on your account
    Connect-PnPOnline -Url $adminUrl -Interactive

    $jsonFilePath = "$($SiteListJsonFile)"
    $sites = Get-Content $jsonFilePath -Raw | ConvertFrom-Json

    $baseUrl = "https://$($TenantOrg).sharepoint.com/sites/"

}
process {

    function GetTenantSite($siteUrl) {
        try{
            $existingSite = Get-PnPTenantSite $siteUrl -ErrorAction SilentlyContinue
        }
        catch {
            Write-Host "  - Site does not exist" -ForegroundColor Yellow
        }
        return $existingSite
    }

    Write-Host "Phase 1 - Create the SharePoint Sites" -ForegroundColor Cyan
    $sites | Foreach-Object {

        $siteUrl = "$($baseUrl)$($_.SiteUrl)"
        $siteTitle = $_.SiteTitle

        # Check for existing site
        $existingSite = GetTenantSite $siteUrl

        if ($existingSite -eq $null) {

            Write-Host "  - Creating new site...."
            New-PnPSite -Type $SiteType -Title $siteTitle -Url $siteUrl -Lcid $_.LocaleId -Owner $OwnerEmail -TimeZone $TimeZoneId -Wait
            Write-Host "  - Created new site"
        }
        else {
            # Site already exists
            Write-Host "  - Site already exists $($siteUrl)" -ForegroundColor Yellow
        }
    }

    # Phase 2 - Create the Hub Sites
    Write-Host "Phase 2 - Create the Hub Sites" -ForegroundColor Cyan
    $sites | Foreach-Object {

        $siteUrl = "$($baseUrl)$($_.SiteUrl)"
        
        # Check for existing site - for those with the hub association only
        $existingSite = GetTenantSite $siteUrl

        if ($existingSite ) {

            if ($_.CreateHubWithName) {

                Write-Host "  - Registering site as Hub site " $siteUrl
                try{
                    Register-PnPHubSite -Site $siteUrl

                    # Update the Hub Title - this can be expanded to include the other options as well
                    Write-Host "  - Updating Hub Title " $siteUrl
                    Set-PnPHubSite -Identity $siteUrl -Title $_.CreateHubWithName

                }catch{
                    Write-Host "  - Site already registered as Hub site" -ForegroundColor Yellow
                }
            }           
        }
        else {
            # Site already exists
            Write-Host "  - Site does not exist" -ForegroundColor Yellow
        }
    }

    Write-Host "Phase 3 - Associate the sites to the Hub Sites" -ForegroundColor Cyan
    $sites | Foreach-Object {

        $siteUrl = "$($baseUrl)$($_.SiteUrl)"
        
        if ($_.JoinHubUrl) {

            # Check for existing site - for those with the hub association only
            $existingSite = GetTenantSite $siteUrl

            if ($existingSite) {

                $joinHubSite = "$($baseUrl)$($_.JoinHubUrl)"
                $hubSite = Get-PnPHubSite -Identity $joinHubSite

                if ($hubsite) {

                    Write-Host "  - Joining site to Hub site " $siteUrl " to " $joinHubSite
                    Add-PnPHubSiteAssociation -Site $siteUrl -HubSite $joinHubSite

                }
                else {
                    # Hubsite not found
                    Write-Host "  - Hub site not found" -ForegroundColor Yellow
                }         
            }
            else {
                # Site already exists
                Write-Host "  - Site does not exist" -ForegroundColor Yellow
            }
        }
    }

    Write-Host "Phase 4 - Associate the sites to the Hub Sites and setup Hub to Hub associations" -ForegroundColor Cyan
    $sites | Foreach-Object {

        $siteUrl = "$($baseUrl)$($_.SiteUrl)"
        
        if ($_.CreateHubWithName -and $_.AssociateHubToHub) {

            # Check for existing site - for those with the hub association only
            $existingSite = GetTenantSite $siteUrl
            
            if ($existingSite) {

                $hubSiteUrl = "$($baseUrl)$($_.AssociateHubToHub)"
                $hubSite = Get-PnPHubSite -Identity $hubSiteUrl

                if ($hubsite) {
            
                    Write-Host "  - Joining site to Hub site to Hub Site" $siteUrl " to " $hubSiteUrl
                    Add-PnPHubToHubAssociation -SourceUrl $siteUrl -TargetUrl $hubSiteUrl

                }
                else {
                    # Hubsite not found
                    Write-Host "  - Hub site not found" -ForegroundColor Yellow
                }
            }
            else {
                # Site already exists
                Write-Host "  - Site does not exist" -ForegroundColor Yellow
            }
        }
    }


    Write-Host "Script Complete! :)" -ForegroundColor Green
}
end {
    #Disconnect-PnPOnline
    Stop-Transcript
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "Base tenant name (e.g., 'contoso')")]
    [string]$TenantOrg,
    
    [Parameter(HelpMessage = "Path to JSON config file")]
    [string]$SiteListJsonFile = "large-intranet-example.json",
    
    [Parameter(HelpMessage = "Time zone ID (default 2 = UTC+01:00)")]
    [int]$TimeZoneId = 2,
    
    [Parameter(HelpMessage = "Locale ID (default 1033 = English)")]
    [int]$LocaleId = 1033,
    
    [Parameter(HelpMessage = "Report only - show what would be created without making changes")]
    [switch]$ReportOnly
)

begin {
    Write-Verbose "Ensuring authentication with CLI for Microsoft 365..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Please run 'm365 login' manually."
    }
    
    if (-not (Test-Path $SiteListJsonFile)) {
        throw "JSON configuration file not found: $SiteListJsonFile"
    }
    
    try {
        $sites = Get-Content $SiteListJsonFile -Raw | ConvertFrom-Json
    }
    catch {
        throw "Failed to parse JSON configuration file: $_"
    }
    
    $baseUrl = "https://$TenantOrg.sharepoint.com/sites/"
    
    Write-Verbose "Retrieving existing sites from tenant..."
    $existingSitesJson = m365 spo site list --type CommunicationSite --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve existing sites: $existingSitesJson"
    }
    $existingSites = @($existingSitesJson | ConvertFrom-Json | Select-Object -ExpandProperty Url)
    
    Write-Verbose "Retrieving existing hub sites from tenant..."
    $existingHubsJson = m365 spo hubsite list --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve existing hub sites: $existingHubsJson"
    }
    $existingHubs = @($existingHubsJson | ConvertFrom-Json)
    
    $script:Summary = @{
        SitesFound = $sites.Count
        SitesAlreadyExist = 0
        SitesCreated = 0
        SitesFailed = 0
        HubsRegistered = 0
        HubsAlreadyExist = 0
        HubsFailed = 0
        SiteAssociations = 0
        SiteAssociationsFailed = 0
        HubAssociations = 0
        HubAssociationsFailed = 0
    }
    
    if ($ReportOnly) {
        Write-Host "`n========================================" -ForegroundColor Yellow
        Write-Host "REPORT-ONLY MODE" -ForegroundColor Yellow
        Write-Host "No changes will be made" -ForegroundColor Yellow
        Write-Host "========================================`n" -ForegroundColor Yellow
        
        Write-Host "Sites to be created/configured:" -ForegroundColor Cyan
        foreach ($site in $sites) {
            $siteUrl = "$baseUrl$($site.SiteUrl)"
            Write-Host "  - $($site.SiteTitle)" -ForegroundColor White
            Write-Host "    URL: $siteUrl" -ForegroundColor Gray
            if ($site.CreateHubWithName) {
                Write-Host "    Hub: $($site.CreateHubWithName)" -ForegroundColor Gray
            }
            if ($site.JoinHubUrl) {
                Write-Host "    Join Hub: $baseUrl$($site.JoinHubUrl)" -ForegroundColor Gray
            }
            if ($site.AssociateHubToHub) {
                Write-Host "    Parent Hub: $baseUrl$($site.AssociateHubToHub)" -ForegroundColor Gray
            }
        }
        Write-Host ""
        return
    }
}

process {
    Write-Host "`nPhase 1 - Create SharePoint Communication Sites" -ForegroundColor Cyan
    
    foreach ($site in $sites) {
        $siteUrl = "$baseUrl$($site.SiteUrl)"
        $siteTitle = $site.SiteTitle
        $lcid = if ($site.LocaleId) { $site.LocaleId } else { $LocaleId }
        
        if ($existingSites -contains $siteUrl) {
            Write-Host "  - Site already exists: $siteTitle" -ForegroundColor Yellow
            Write-Host "    $siteUrl" -ForegroundColor Gray
            $Summary.SitesAlreadyExist++
            continue
        }
        
        if ($PSCmdlet.ShouldProcess($siteUrl, "Create communication site")) {
            Write-Host "  - Creating site: $siteTitle" -ForegroundColor White
            
            try {
                m365 spo site add --type CommunicationSite --url $siteUrl --title $siteTitle --lcid $lcid --timeZone $TimeZoneId --wait --output json 2>&1 | Out-Null
                
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to create site: $siteTitle ($siteUrl)"
                    $Summary.SitesFailed++
                    continue
                }
                
                Write-Host "    Created successfully" -ForegroundColor Green
                $Summary.SitesCreated++
            }
            catch {
                Write-Warning "Failed to create site: $siteTitle ($siteUrl). Error: $_"
                $Summary.SitesFailed++
                continue
            }
        }
    }
    
    Write-Host "`nPhase 2 - Register Hub Sites" -ForegroundColor Cyan
    
    foreach ($site in $sites) {
        if (-not $site.CreateHubWithName) {
            continue
        }
        
        $siteUrl = "$baseUrl$($site.SiteUrl)"
        $hubTitle = $site.CreateHubWithName
        
        $existingHub = $existingHubs | Where-Object { $_.SiteUrl -eq $siteUrl }
        
        if ($PSCmdlet.ShouldProcess($siteUrl, "Register as hub site")) {
            if ($existingHub) {
                Write-Host "  - Hub already registered: $hubTitle" -ForegroundColor Yellow
                Write-Host "    $siteUrl" -ForegroundColor Gray
                $Summary.HubsAlreadyExist++
                continue
            }
            
            Write-Host "  - Registering hub: $hubTitle" -ForegroundColor White
            try {
                $hubResult = m365 spo hubsite register --siteUrl $siteUrl --output json 2>&1
                
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to register hub: $hubTitle ($siteUrl)"
                    $Summary.HubsFailed++
                    continue
                }
                
                $hubData = $hubResult | ConvertFrom-Json
                $hubId = $hubData.ID
                
                $existingHubs += $hubData
                
                Write-Verbose "Updating hub title to: $hubTitle"
                m365 spo hubsite set --id $hubId --title $hubTitle --output json 2>&1 | Out-Null
                
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Hub registered but failed to update title: $hubTitle"
                }
                
                Write-Host "    Registered successfully" -ForegroundColor Green
                $Summary.HubsRegistered++
            }
            catch {
                Write-Warning "Failed to register hub: $hubTitle ($siteUrl). Error: $_"
                $Summary.HubsFailed++
                continue
            }
        }
    }
    
    Write-Host "`nPhase 3 - Associate Sites to Hub Sites" -ForegroundColor Cyan
    
    foreach ($site in $sites) {
        if (-not $site.JoinHubUrl) {
            continue
        }
        
        $siteUrl = "$baseUrl$($site.SiteUrl)"
        $hubUrl = "$baseUrl$($site.JoinHubUrl)"
        
        if ($PSCmdlet.ShouldProcess($siteUrl, "Associate to hub site $hubUrl")) {
            Write-Host "  - Associating: $($site.SiteTitle)" -ForegroundColor White
            Write-Host "    to hub: $hubUrl" -ForegroundColor Gray
            
            try {
                $hubData = $existingHubs | Where-Object { $_.SiteUrl -eq $hubUrl }
                
                if (-not $hubData) {
                    Write-Warning "Hub site not found: $hubUrl"
                    $Summary.SiteAssociationsFailed++
                    continue
                }
                
                $hubId = $hubData.ID
                
                m365 spo site hubsite connect --siteUrl $siteUrl --id $hubId 2>&1 | Out-Null
                
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to associate site: $($site.SiteTitle) to hub: $hubUrl"
                    $Summary.SiteAssociationsFailed++
                    continue
                }
                
                Write-Host "    Associated successfully" -ForegroundColor Green
                $Summary.SiteAssociations++
            }
            catch {
                Write-Warning "Failed to associate site: $($site.SiteTitle). Error: $_"
                $Summary.SiteAssociationsFailed++
                continue
            }
        }
    }
    
    Write-Host "`nPhase 4 - Associate Hub Sites to Parent Hub Sites" -ForegroundColor Cyan
    
    foreach ($site in $sites) {
        if (-not $site.AssociateHubToHub) {
            continue
        }
        
        $childHubUrl = "$baseUrl$($site.SiteUrl)"
        $parentHubUrl = "$baseUrl$($site.AssociateHubToHub)"
        
        Write-Verbose "Associating hub to parent hub: $childHubUrl -> $parentHubUrl"
        
        if ($PSCmdlet.ShouldProcess($childHubUrl, "Associate hub to parent hub $parentHubUrl")) {
            Write-Host "  - Associating hub: $($site.SiteTitle)" -ForegroundColor White
            Write-Host "    to parent hub: $parentHubUrl" -ForegroundColor Gray
            
            try {
                m365 spo hubsite connect --url $childHubUrl --parentUrl $parentHubUrl 2>&1 | Out-Null
                
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to associate hub: $($site.SiteTitle) to parent hub: $parentHubUrl"
                    $Summary.HubAssociationsFailed++
                    continue
                }
                
                Write-Host "    Associated successfully" -ForegroundColor Green
                $Summary.HubAssociations++
            }
            catch {
                Write-Warning "Failed to associate hub: $($site.SiteTitle). Error: $_"
                $Summary.HubAssociationsFailed++
                continue
            }
        }
    }
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "EXECUTION SUMMARY" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    
    Write-Host "Sites:" -ForegroundColor White
    Write-Host "  Total Found: $($Summary.SitesFound)" -ForegroundColor White
    if ($Summary.SitesAlreadyExist -gt 0) {
        Write-Host "  Already Exist: " -NoNewline
        Write-Host $Summary.SitesAlreadyExist -ForegroundColor Yellow
    }
    Write-Host "  Created: " -NoNewline
    Write-Host $Summary.SitesCreated -ForegroundColor Green
    if ($Summary.SitesFailed -gt 0) {
        Write-Host "  Failed: " -NoNewline
        Write-Host $Summary.SitesFailed -ForegroundColor Red
    }
    
    Write-Host "`nHub Sites:" -ForegroundColor White
    if ($Summary.HubsAlreadyExist -gt 0) {
        Write-Host "  Already Exist: " -NoNewline
        Write-Host $Summary.HubsAlreadyExist -ForegroundColor Yellow
    }
    Write-Host "  Registered: " -NoNewline
    Write-Host $Summary.HubsRegistered -ForegroundColor Green
    if ($Summary.HubsFailed -gt 0) {
        Write-Host "  Failed: " -NoNewline
        Write-Host $Summary.HubsFailed -ForegroundColor Red
    }
    
    Write-Host "`nSite Associations:" -ForegroundColor White
    Write-Host "  Created: " -NoNewline
    Write-Host $Summary.SiteAssociations -ForegroundColor Green
    if ($Summary.SiteAssociationsFailed -gt 0) {
        Write-Host "  Failed: " -NoNewline
        Write-Host $Summary.SiteAssociationsFailed -ForegroundColor Red
    }
    
    Write-Host "`nHub-to-Hub Associations:" -ForegroundColor White
    Write-Host "  Created: " -NoNewline
    Write-Host $Summary.HubAssociations -ForegroundColor Green
    if ($Summary.HubAssociationsFailed -gt 0) {
        Write-Host "  Failed: " -NoNewline
        Write-Host $Summary.HubAssociationsFailed -ForegroundColor Red
    }
    
    Write-Host "`n========================================`n" -ForegroundColor Cyan
}

# Example 1: Basic usage
# .\Create-MultiHubSites.ps1 -TenantOrg "contoso" -SiteListJsonFile "large-intranet-example.json"

# Example 2: Report-only mode (preview without making changes)
# .\Create-MultiHubSites.ps1 -TenantOrg "contoso" -SiteListJsonFile "large-intranet-example.json" -ReportOnly

# Example 3: WhatIf mode (PowerShell built-in confirmation)
# .\Create-MultiHubSites.ps1 -TenantOrg "contoso" -SiteListJsonFile "large-intranet-example.json" -WhatIf

# Example 4: Verbose output for detailed progress
# .\Create-MultiHubSites.ps1 -TenantOrg "contoso" -SiteListJsonFile "large-intranet-example.json" -Verbose

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [JSON](#tab/json)
```json

[
    {
        "SiteTitle":"Ketchup Inc",
        "SiteUrl":"ketchupinc-intranet",
        "CreateHubWithName": "Ketchup Inc Intranet",
        "JoinHubUrl": "",
        "AssociateHubToHub": "",
        "LocaleId":1033
    },
    {
        "SiteTitle":"Ketchup Inc - News",
        "SiteUrl":"ketchupinc-news",
        "CreateHubWithName": "",
        "JoinHubUrl": "ketchupinc-intranet",
        "AssociateHubToHub": "",
        "LocaleId":1033
    },


    {
        "SiteTitle":"Ketchup Inc - HR Hub",
        "SiteUrl":"ketchupinc-hr",
        "CreateHubWithName": "Ketchup Inc HR Hub",
        "JoinHubUrl": "",
        "AssociateHubToHub": "ketchupinc-intranet",
        "LocaleId":1033
    },
    {
        "SiteTitle":"Ketchup Inc - HR Support",
        "SiteUrl":"ketchupinc-hr-support",
        "CreateHubWithName": "",
        "JoinHubUrl": "ketchupinc-hr",
        "AssociateHubToHub": "",
        "LocaleId":1033
    },
    {
        "SiteTitle":"Ketchup Inc - HR Management",
        "SiteUrl":"ketchupinc-hr-management",
        "CreateHubWithName": "",
        "JoinHubUrl": "ketchupinc-hr",
        "AssociateHubToHub": "",
        "LocaleId":1033
    },


    {
        "SiteTitle":"Ketchup Inc - IT Hub",
        "SiteUrl":"ketchupinc-it",
        "CreateHubWithName": "Ketchup Inc IT Hub",
        "JoinHubUrl": "",
        "AssociateHubToHub": "ketchupinc-intranet",
        "LocaleId":1033
    },
    {
        "SiteTitle":"Ketchup Inc - IT Services",
        "SiteUrl":"ketchupinc-it-services",
        "CreateHubWithName": "",
        "JoinHubUrl": "ketchupinc-it",
        "AssociateHubToHub": "",
        "LocaleId":1033
    },
    {
        "SiteTitle":"Ketchup Inc - IT Support",
        "SiteUrl":"ketchupinc-it-support",
        "CreateHubWithName": "",
        "JoinHubUrl": "ketchupinc-it",
        "AssociateHubToHub": "",
        "LocaleId":1033
    },
    {
        "SiteTitle":"Ketchup Inc - IT Training",
        "SiteUrl":"ketchupinc-it-training",
        "CreateHubWithName": "",
        "JoinHubUrl": "ketchupinc-it",
        "AssociateHubToHub": "",
        "LocaleId":1033
    }
]

```
***


## Contributors

| Author(s) |
|-----------|
| Paul Bullock |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-create-multi-hub-sites" aria-hidden="true" />

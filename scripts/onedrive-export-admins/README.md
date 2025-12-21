

# Export OneDrive Admins

## Summary
Have you ever needed to know which Admins have added themselves to which OneDrives? This script exports every OneDrive in the tenant, and the site collection admins of the site. This helps audit which admins have unnecessary access to user OneDrives. Once you have the report, you can identify unnecessary access by filtering in Excel. This sample is available in both PnP PowerShell and CLI for Microsoft 365.

![Example Screenshot](assets/OneDriveAdmins.png)

The report produces a csv file with one row per Site Collection Admin and OneDrive. This report has four columns:
SiteURL
SiteName
SiteCollectionAdmin
SiteCollectionAdminName


# [PnP PowerShell](#tab/pnpps)

```powershell

#Parameters
$AdminURL = "https://contoso-admin.sharepoint.com"
$ReportOutput = "OneDriveAdmins.csv"

#Authentication Details - If you have not registered PnP before, simply run the command Register-PnPAzureADApp to create an App
$ClientId = "xxxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxx"
$Thumbprint = "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
$Tenant = "contoso.onmicrosoft.com"

#Connect to SharePoint Online Admin site
Connect-PnPOnline $AdminURL -ClientId $ClientId -Thumbprint $Thumbprint  -Tenant $Tenant 

#Get all Mysites
$MySites = Get-PnPTenantSite -IncludeOneDriveSites -Filter "Url -like '-my.sharepoint.com/personal/'"

foreach ($MySite in $MySites) {
    try{
        write-host "Processing"$Mysite.Title -ForegroundColor Green
        #Connect to the MySite
        Connect-PnPOnline $MySite.Url -ClientId $ClientId -Thumbprint $Thumbprint  -Tenant $Tenant -ErrorAction Stop
        #Get the admins
        $Admins = Get-PnPSiteCollectionAdmin -ErrorAction Stop
        
        foreach($admin in $Admins){
            #Foreach admin make a record to output to CSV   
            $Result = New-Object PSObject -Property ([ordered]@{
                SiteURL = $Mysite.Url
                SiteName = $Mysite.Title
                SiteCollectionAdmin = $admin.Email
                SiteCollectionAdminName = $admin.Title
            })
            
            #Export the results to CSV
            $Result | Export-Csv -Path $ReportOutput -NoTypeInformation -Append

        }
    }catch{
        #We encountered an error, print it to the screen
        write-host "Error with site collection"$Mysite.Title -ForegroundColor Red
    }
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(HelpMessage = "Path where the CSV file will be saved")]
    [string]$OutputPath = "OneDriveAdmins_$(Get-Date -Format 'yyyy-MM-dd-HHmmss').csv",
    
    [Parameter(HelpMessage = "Export results to CSV file")]
    [switch]$ExportToCsv
)

begin {
    Write-Verbose "Starting OneDrive Admin export script"
    
    if ($ExportToCsv) {
        $directory = Split-Path -Path $OutputPath -Parent
        if ($directory -and -not (Test-Path -Path $directory)) {
            throw "Directory does not exist: $directory"
        }
    }
    
    Write-Verbose "Verifying CLI for Microsoft 365 connection"
    m365 login --ensure 2>&1 | Out-Null
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to verify CLI for Microsoft 365 connection. Please run 'm365 login' first."
    }
    Write-Verbose "CLI for Microsoft 365 connection verified"
    
    $script:Summary = @{
        OneDriveSitesFound = 0
        AdminsFound = 0
        Failures = 0
    }
    
    $script:Report = [System.Collections.ArrayList]::new()
}

process {
    Write-Verbose "Retrieving all OneDrive sites from tenant"
    $sitesJson = m365 spo site list --withOneDriveSites --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve OneDrive sites. CLI output: $sitesJson"
    }
    
    $allSites = @($sitesJson | ConvertFrom-Json)
    Write-Verbose "Found $($allSites.Count) total site(s)"
    
    $oneDriveSites = $allSites | Where-Object { $_.Url -like '*-my.sharepoint.com/personal/*' }
    $script:Summary.OneDriveSitesFound = $oneDriveSites.Count
    Write-Verbose "Found $($oneDriveSites.Count) OneDrive site(s)"
    
    if ($oneDriveSites.Count -eq 0) {
        Write-Warning "No OneDrive sites found in the tenant"
    }

    if ($oneDriveSites.Count -gt 0) {
        foreach ($site in $oneDriveSites) {
            Write-Verbose "Processing OneDrive: $($site.Title)"
            
            try {
                if ($PSCmdlet.ShouldProcess($site.Url, 'Get site collection admins')) {
                    $adminsJson = m365 spo site admin list --siteUrl $site.Url --asAdmin --output json 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to retrieve admins for OneDrive '$($site.Title)'. CLI: $adminsJson"
                        $script:Summary.Failures++
                        continue
                    }
                    
                    $admins = @($adminsJson | ConvertFrom-Json)
                    Write-Verbose "  Found $($admins.Count) admin(s)"
                    
                    foreach ($admin in $admins) {
                        [void]$script:Report.Add([PSCustomObject]@{
                            SiteURL = $site.Url
                            SiteName = $site.Title ?? "(No Title)"
                            SiteCollectionAdmin = $admin.Email ?? $admin.LoginName
                            SiteCollectionAdminName = $admin.Title ?? "(No Display Name)"
                            IsPrimaryAdmin = $admin.IsPrimaryAdmin ?? $false
                        })
                        $script:Summary.AdminsFound++
                    }
                }
            } catch {
                Write-Warning "Error processing OneDrive '$($site.Title)': $($_.Exception.Message)"
                $script:Summary.Failures++
            }
        }
    }
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "OneDrive Admin Export Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "OneDrive sites found: $($script:Summary.OneDriveSitesFound)" -ForegroundColor White
    Write-Host "Total admins found:   $($script:Summary.AdminsFound)" -ForegroundColor Green
    Write-Host "Failures:             $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "========================================`n" -ForegroundColor Cyan
    
    if ($ExportToCsv -and $script:Report.Count -gt 0) {
        if ($PSCmdlet.ShouldProcess($OutputPath, 'Export results to CSV')) {
            $script:Report | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
            Write-Host "Results exported to: $OutputPath" -ForegroundColor Green
        }
    } elseif ($script:Report.Count -gt 0) {
        Write-Host "OneDrive Administrators:" -ForegroundColor Yellow
        $script:Report | Format-Table -AutoSize
    }
    
    Write-Verbose "Script execution completed"
}

# Usage examples:
# .\Export-OneDriveAdmins.ps1 -ExportToCsv -Verbose
# .\Export-OneDriveAdmins.ps1
# .\Export-OneDriveAdmins.ps1 -OutputPath "C:\Reports\OneDriveAdmins.csv" -ExportToCsv
# .\Export-OneDriveAdmins.ps1 -WhatIf
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| Matt Maher |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it)|


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/onedrive-export-admins" aria-hidden="true" />

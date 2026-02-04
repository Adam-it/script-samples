

# Update User Profile Properties

## Summary

This script shows how to update user profile properties using CLI for Microsoft 365 or PnP PowerShell.

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as below,
- First, we will connect to a SharePoint Admin tenant.
	- then we will ask a user to enter location and skills (We can ask properties as per our requirements).
    - And then we will update current user profile properties.
 
# [PnP PowerShell](#tab/pnpps)
```powershell

$adminSiteURL = "https://domain-admin.sharepoint.com/"
$username = "chandani@domain.onmicrosoft.com"
$password = "********"
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$Creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd

Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $Creds)
     
    Write-Host "Connecting to Tenant Admin Site '$($adminSiteURL)'" -f Yellow   
    Connect-PnPOnline -Url $adminSiteURL -Credentials $Creds
    Write-Host "Connection Successful" -f Green 
}

Function UpdateUserProfileProperties {
    try {
        $Location = Read-Host "Enter location" 
        $Skills = Read-Host "Enter skills by comma seprated(e.g. SPFx, PS)"          
        Write-Host "Updating user profile Properties for:" $username -f Yellow        
        Set-PnPUserProfileProperty -Account $username -PropertyName 'SPS-Location' -Value $Location 
        Set-PnPUserProfileProperty -Account $username -PropertyName "SPS-Skills" -Value $Skills       
        Write-Host "Updated user profile Properties for:" $username -f Green 
    }
    catch {
        Write-Host "Getting error in updating user profile Propertiese:" $_.Exception.Message -ForegroundColor Red                 
    }  
}

Function StartProcessing {
    Login($Creds);
    UpdateUserProfileProperties
}

StartProcessing

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, HelpMessage = "User principal name (email address)")]
    [ValidatePattern('^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')]
    [string]$UserName,
    
    [Parameter(Mandatory = $false, HelpMessage = "Location value for SPS-Location property")]
    [string]$Location,
    
    [Parameter(Mandatory = $false, HelpMessage = "Comma-separated skills for SPS-Skills property")]
    [string]$Skills,
    
    [Parameter(Mandatory = $false, HelpMessage = "Output path for transcript log")]
    [ValidateScript({ Test-Path -Path $_ -PathType Container })]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $script:Summary = @{
        PropertiesUpdated = 0
        Failures = 0
    }
    
    $timestamp = Get-Date -Format "yyyy-MM-dd-HHmmss"
    $transcriptPath = Join-Path $OutputPath "cli-userprofile-log-$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Connecting to Microsoft 365..." -ForegroundColor Yellow
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to authenticate with Microsoft 365"
    }
    
    Write-Host "Connection successful!" -ForegroundColor Green
}

process {
    $propertiesToUpdate = [System.Collections.ArrayList]::new()
    
    if ([string]::IsNullOrEmpty($Location)) {
        $Location = Read-Host "Enter location (or press Enter to skip)"
    }
    if (-not [string]::IsNullOrEmpty($Location)) {
        $null = $propertiesToUpdate.Add(@{
            Name = 'SPS-Location'
            Value = $Location
        })
    }
    
    if ([string]::IsNullOrEmpty($Skills)) {
        $Skills = Read-Host "Enter skills (comma-separated, or press Enter to skip)"
    }
    if (-not [string]::IsNullOrEmpty($Skills)) {
        $null = $propertiesToUpdate.Add(@{
            Name = 'SPS-Skills'
            Value = $Skills
        })
    }
    
    if ($propertiesToUpdate.Count -eq 0) {
        Write-Warning "No properties to update. Please provide at least one property value."
        return
    }
    
    Write-Host "`nUpdating user profile properties for: $UserName" -ForegroundColor Yellow
    
    foreach ($property in $propertiesToUpdate) {
        try {
            if ($PSCmdlet.ShouldProcess($UserName, "Update property '$($property.Name)' to '$($property.Value)'")) {
                Write-Verbose "Updating property: $($property.Name)"
                
                m365 spo userprofile set --userName $UserName --propertyName $property.Name --propertyValue $property.Value
                
                if ($LASTEXITCODE -ne 0) {
                    throw "CLI command failed with exit code $LASTEXITCODE"
                }
                
                Write-Host "  ✓ Successfully updated $($property.Name)" -ForegroundColor Green
                $script:Summary.PropertiesUpdated++
            }
        }
        catch {
            Write-Warning "Failed to update property '$($property.Name)': $_"
            $script:Summary.Failures++
            continue
        }
    }
}

end {
    Stop-Transcript
    
    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "User: $UserName" -ForegroundColor Gray
    Write-Host "Properties Updated: $($Summary.PropertiesUpdated)" -ForegroundColor Green
    Write-Host "Failures: $($Summary.Failures)" -ForegroundColor $(if ($Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "Transcript: $transcriptPath" -ForegroundColor Gray
    Write-Host "============================`n" -ForegroundColor Cyan
}

# Update both Location and Skills properties
# .\Update-UserProfile.ps1 -UserName "john.doe@contoso.com" -Location "London" -Skills "PowerShell, Azure"

# Test changes without executing (WhatIf mode)
# .\Update-UserProfile.ps1 -UserName "john.doe@contoso.com" -Location "New York" -Skills "SharePoint, Teams" -WhatIf

# Interactive mode - prompts for property values
# .\Update-UserProfile.ps1 -UserName "john.doe@contoso.com"

# Update with verbose logging
# .\Update-UserProfile.ps1 -UserName "john.doe@contoso.com" -Location "Paris" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Chandani Prajapati |
| [Jasey Waegebaert](https://github.com/Jwaegebaert) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-user-profile-properties" aria-hidden="true" />

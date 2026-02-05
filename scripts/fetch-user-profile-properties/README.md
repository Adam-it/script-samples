

# Fetch User Profile Properties From Site Collection And Export To CSV

## Summary

This script shows how to get users or user profile properties from any SharePoint site collection using CLI for Microsoft 365 or PnP PowerShell and export them to CSV or Excel format.

![Example Screenshot](assets/example.png)

## Implementation
 
Open Windows Powershell ISE
Create a new file and write a script
 
Now we will see all the steps which we required to achieve the solution:

1. We will read the site URL from the user
2. then we will connect to the O365 admin site and then we will connect to the site which the user has entered
3. Create a function to bind a CSV
4. Create a function to get user profile properties by email d
5. In the main function we will write a logic to get web and users of site collection URL and then get all the properties and bind it to CSV

So at the end, our script will be like this,

# [PnP PowerShell](#tab/pnpps)

```powershell

$basePath = #base path where you want to save CSV file("D:\Chandani\...\")
$dateTime = "{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$csvPath = $basePath + "\userdetails" + $dateTime + ".csv"
$adminSiteURL = "https://****-admin.sharepoint.com/" #O365 admin site URL
$username = #user email id
$password = "********"
$secureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$Creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd
$global:userDetails = @()
$index = 1;
$userInfo;
  
Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $Creds)
 
    #connect to O365 admin site
    Write-Host "Connecting to Tenant Admin Site '$($adminSiteURL)'" -f Yellow | Out-File $LogFile -Append -Force
  
    Connect-PnPOnline -Url $adminSiteURL -Credentials $Creds
    Write-Host "Connection Successful" -f Yellow | Out-File $LogFile -Append -Force
   
}
Function StartProcessing {
    Login($Creds);
    ConnectionToSite($Creds)
}

Function ConnectionToSite() {
    $siteURL = Read-Host "Please enter site collection URL" 
  
    try {            
        Write-Host "Connecting to Site '$($siteURL)'" -f Yellow          
                              
        $SCWeb = Get-PnPWeb -Identity ""              
                                                     
        $getusers = Get-PnPUser -Web $SCWeb

        ForEach ($user in $getusers) { 
            $email = $user.Email
            If ($email) {
                $userInfo = GetUserProfileProperties $email        
                #creating object fro CSV
                $global:userDetails += New-Object PSObject -Property ([ordered]@{                   
                        Id            = $index
                        GUID          = $userInfo.'UserProfile_GUID'
                        FirstName     = $userInfo.FirstName
                        LastName      = $userInfo.LastName
                        WorkEmail     = $userInfo.WorkEmail 
                        PictureURL    = $userInfo.PictureURL    
                        Department    = $userInfo.Department
                        PreferredName = $userInfo.PreferredName                        
                    })
                $index++ 
            } 
        }                                                                
    }
    catch {
        Write-Host -f Red "Error in connecting to Site '$($TenantSite)'"                        
    }                                     
    BindingtoCSV($global:userDetails) 
}

Function BindingtoCSV {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $Global)   
    Write-Host -f Yellow "Exporting to CSV..."
    $userDetails | Export-Csv $csvPath -NoTypeInformation -Append
    Write-Host -f Yellow "Exported Successfully..."
}

Function GetUserProfileProperties($username) {
    $Properties = Get-PnPUserProfileProperty -Account $username
    $Properties = $Properties.UserProfileProperties
   
    If ($Properties) {
        $Properties = $Properties
    }
    else {
        $Properties = $null
    }
    return $Properties
}

StartProcessing

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Web URL from which to retrieve users")]
    [ValidatePattern('^https://.*\.sharepoint\.(com|us|mil|cn)')]
    [string]$WebUrl,
    
    [Parameter(Mandatory = $false, HelpMessage = "Output path for CSV export and transcript")]
    [ValidateScript({ Test-Path -Path $_ -PathType Container })]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    $script:Summary = @{
        UsersFound = 0
        UsersProcessed = 0
        Failures = 0
    }
    
    $timestamp = Get-Date -Format "yyyy-MM-dd-HHmmss"
    $csvPath = Join-Path $OutputPath "userdetails-$timestamp.csv"
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
    Write-Host "`nRetrieving users from $WebUrl..." -ForegroundColor Yellow
    Write-Progress -Activity "Fetching User Profiles" -Status "Retrieving users from site" -PercentComplete 0
    
    $users = m365 spo user list --webUrl $WebUrl --output json | ConvertFrom-Json
    
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve users from site"
    }
    
    $usersWithMail = @($users | Where-Object { -not [string]::IsNullOrEmpty($_.Email) })
    $script:Summary.UsersFound = $usersWithMail.Count
    
    if ($usersWithMail.Count -eq 0) {
        Write-Host "No users with email addresses found in the site" -ForegroundColor Yellow
        return
    }
    
    Write-Host "Found $($usersWithMail.Count) user(s) with email addresses" -ForegroundColor Cyan
    Write-Host "Fetching user profile properties..." -ForegroundColor Yellow
    
    $userDetailsArray = [System.Collections.ArrayList]::new()
    
    for ($i = 0; $i -lt $usersWithMail.Count; $i++) {
        $user = $usersWithMail[$i]
        $userMail = $user.Email
        $percentComplete = (($i + 1) / $usersWithMail.Count) * 100
        
        Write-Progress -Activity "Fetching User Profiles" -Status "Processing $($i + 1)/$($usersWithMail.Count): $userMail" -PercentComplete $percentComplete
        Write-Verbose "Processing user: $userMail"
        
        try {
            $userProfile = m365 spo userprofile get --userName $userMail --output json | ConvertFrom-Json
            
            if ($LASTEXITCODE -ne 0) {
                throw "CLI command failed with exit code $LASTEXITCODE"
            }
            
            if ($userProfile.Email) {
                $userProfileProperties = $userProfile.UserProfileProperties
                
                $null = $userDetailsArray.Add([PSCustomObject][ordered]@{
                    Id            = $i + 1
                    GUID          = ($userProfileProperties | Where-Object { $_.Key -eq 'UserProfile_GUID' }).Value
                    FirstName     = ($userProfileProperties | Where-Object { $_.Key -eq 'FirstName' }).Value
                    LastName      = ($userProfileProperties | Where-Object { $_.Key -eq 'LastName' }).Value
                    WorkEmail     = ($userProfileProperties | Where-Object { $_.Key -eq 'WorkEmail' }).Value
                    PictureURL    = ($userProfileProperties | Where-Object { $_.Key -eq 'PictureURL' }).Value
                    Department    = ($userProfileProperties | Where-Object { $_.Key -eq 'Department' }).Value
                    PreferredName = ($userProfileProperties | Where-Object { $_.Key -eq 'PreferredName' }).Value
                })
                
                $script:Summary.UsersProcessed++
            }
            else {
                Write-Warning "User profile for $userMail returned no email. Possible external user or group email"
                $script:Summary.Failures++
            }
        }
        catch {
            Write-Warning "Failed to retrieve profile for user $userMail: $_"
            $script:Summary.Failures++
            continue
        }
    }
    
    Write-Progress -Activity "Fetching User Profiles" -Completed
    
    if ($userDetailsArray.Count -gt 0) {
        Write-Host "`nExporting to CSV..." -ForegroundColor Yellow
        $userDetailsArray | Export-Csv -Path $csvPath -NoTypeInformation
        Write-Host "Exported $($userDetailsArray.Count) user profile(s) to: $csvPath" -ForegroundColor Green
    }
    else {
        Write-Host "`nNo user profiles were successfully retrieved" -ForegroundColor Yellow
    }
}

end {
    Stop-Transcript
    
    Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
    Write-Host "Users Found: $($Summary.UsersFound)" -ForegroundColor Green
    Write-Host "Users Processed: $($Summary.UsersProcessed)" -ForegroundColor Green
    Write-Host "Failures: $($Summary.Failures)" -ForegroundColor $(if ($Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "CSV Export: $csvPath" -ForegroundColor Gray
    Write-Host "Transcript: $transcriptPath" -ForegroundColor Gray
    Write-Host "============================`n" -ForegroundColor Cyan
}

# Basic usage
# .\Fetch-User-Profile-Properties.ps1 -WebUrl "https://contoso.sharepoint.com/sites/Intranet"

# Specify custom output path
# .\Fetch-User-Profile-Properties.ps1 -WebUrl "https://contoso.sharepoint.com/sites/Intranet" -OutputPath "C:\Reports"

# With verbose logging
# .\Fetch-User-Profile-Properties.ps1 -WebUrl "https://contoso.sharepoint.com/sites/Intranet" -Verbose

# Process multiple sites in pipeline
# @("https://contoso.sharepoint.com/sites/Site1", "https://contoso.sharepoint.com/sites/Site2") | ForEach-Object { .\Fetch-User-Profile-Properties.ps1 -WebUrl $_ }
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Source Credit

Sample first appeared on [Fetch User Profile Properties From Site Collection And Export To CSV Using PNP PowerShell | Microsoft 365 PnP Blog](https://techcommunity.microsoft.com/t5/microsoft-365-pnp-blog/fetch-user-profile-properties-from-site-collection-and-export-to/ba-p/2232136)

## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati |
| Mathijs Verbeeck |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/fetch-user-profile-properties" aria-hidden="true" />



# Add Bulk Users to SharePoint Site Groups

## Summary

This sample shows how to add bulk users to SharePoint site groups from CSV.

## Bulk Users CSV

| SiteURL                                                     | GroupName        | Users                           |
| -------                                                     | ---------        | ------                          |
| https://domain.sharepoint.com/sites/SPFxLearning/           | SPFx Users       | user2@domain.onmicrosoft.com    |
| https://domain.sharepoint.com/sites/PowerShellLearning/     | PowerShell Users | chandani@domain.onmicrosoft.com |
| https://domain.sharepoint.com/sites/PowerPlatformLearning/  | Power Users | chandani@domain.onmicrosoft.com |

You can download input CSV reference file at [here](assets/DummyInput.csv).

![Example Screenshot](assets/preview.png)

## Implementation

1. Open Windows PowerShell ISE
2. Create a new file and write a script 
3. Now we will see all the steps which are required to achieve the solution:
   - Create a function to read a CSV file and store it in a global variable.
   - Create a function to connect the M365 admin site.
   - Create a function to add users to a group, in this first we will be looping all elements and connecting to the particular site. After that, we will check if the current user exists in the current group. If not exists then we will add it.

# [PnP PowerShell](#tab/pnpps)

```powershell
$AdminSiteURL = "https://domain-admin.sharepoint.com/"
$Username = "chandani@domain.onmicrosoft.com"
$Password = "********"
$SecureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$Creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd
$CSVPath = "E:\Contribution\PnP-Scripts\bulk-add-users-to-group\SP-Usres.csv"
$global:CSVData = @()

Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $Creds)     
    Write-Host "Connecting to Tenant Admin Site '$($AdminSiteURL)'" -ForegroundColor Yellow   
    Connect-PnPOnline -Url $AdminSiteURL -Credentials $Creds
    Write-Host "Connection Successful!" -ForegroundColor Green 
    ReadCSVFile
}

Function ReadCSVFile() {
    Write-Host "Reading CSV file..." -ForegroundColor Yellow   
    $global:CSVData = Import-Csv $CSVPath
    Write-Host "Reading CSV file successfully!" -ForegroundColor Green   
    AddUsersToGroups
}

Function AddUsersToGroups() {
    ForEach ($CurrentItem in $CSVData) {
        Try {
            #Connect to SharePoint Online Site
            Write-host "Connecting to Site: "$CurrentItem.SiteURL
            Connect-PnPOnline -Url $CurrentItem.SiteURL -Credentials $Creds
  
            #Get the group
            $Group = Get-PnPGroup -Identity $CurrentItem.GroupName
  
            #Get group members
            $GroupMembers = Get-PnPGroupMembers -Identity $Group | select Email
            
            #Check if user is exists in a group or not
            $IsUserExists = $GroupMembers -match $CurrentItem.Users
            if ($IsUserExists.Length) {
                Write-Host "User $($CurrentItem.Users) is already exists in $($Group.Title)" -ForegroundColor Yellow                
            }
            else {
                Write-Host "Adding User $($CurrentItem.Users) to $($Group.Title)" -ForegroundColor Yellow  
                Add-PnPGroupMember -LoginName $CurrentItem.Users -Identity $Group
                Write-host "Added User $($CurrentItem.Users) to $($Group.Title)" -ForegroundColor Green
            }                        
        }
        Catch {
            write-host "Error Adding User to Group:" $_.Exception.Message -ForegroundColor Red 
        }
    }
}

Function StartProcessing {
    Login($Creds);    
}

StartProcessing
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [SPO Management Shell](#tab/spoms-ps)

```powershell
$AdminSiteURL = "https://domain-admin.sharepoint.com/"
$Username = "chandani@domain.onmicrosoft.com"
$Password = "********"
$SecureStringPwd = $password | ConvertTo-SecureString -AsPlainText -Force 
$Creds = New-Object System.Management.Automation.PSCredential -ArgumentList $username, $secureStringPwd
$CSVPath = "E:\Contribution\PnP-Scripts\bulk-add-users-to-group\SP_Dummy_Users.csv"
$global:CSVData = @()

Function Login() {
    [cmdletbinding()]
    param([parameter(Mandatory = $true, ValueFromPipeline = $true)] $Creds)     
    Write-Host "Connecting to Tenant Admin Site '$($AdminSiteURL)'" -f Yellow   
    Connect-SPOService -Url $AdminSiteURL -Credential $Creds
    Write-Host "Connection Successful!" -f Green 
    ReadCSVFile
}

Function ReadCSVFile() {
    Write-Host "Reading CSV file..." -ForegroundColor Yellow   
    $global:CSVData = Import-Csv $CSVPath
    Write-Host "Reading CSV file successfully!" -ForegroundColor Green   
    AddUsersToGroups
}

Function AddUsersToGroups() {
    ForEach ($CurrentItem in $CSVData) {
        Try {
            #Connect to SharePoint Online Site
            Write-host "Connecting to Site: "$CurrentItem.SiteURL
            $Site = Get-SPOSite -Identity $CurrentItem.SiteURL
  
            #Get group members
            $GroupMembers = Get-SPOUser -Site $CurrentItem.SiteURL -Group $CurrentItem.GroupName | select Email
            $IsUserExists = $GroupMembers -match $CurrentItem.Users
            if ($IsUserExists.Length) {
                Write-Host "User $($CurrentItem.Users) is already exists in $($Group.Title)" -ForegroundColor Yellow                
            }
            else {
                Write-Host "Adding User $($CurrentItem.Users) to $($CurrentItem.GroupName)" -ForegroundColor Yellow  
                Add-SPOUser -LoginName $CurrentItem.Users -Group  $CurrentItem.GroupName -Site $CurrentItem.SiteURL
                Write-host "Added User $($CurrentItem.Users) to $($CurrentItem.GroupName)" -ForegroundColor Green
            }                        
        }
        Catch {
            write-host -f Red "Error Adding User to Group:" $_.Exception.Message
        }
    }
}

Function StartProcessing {
    Login($Creds); 
}

StartProcessing
```

[!INCLUDE [More about SPO Management Shell](../../docfx/includes/MORE-SPOMS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
# .\Add-BulkUsers.ps1 -CsvPath ".\assets\DummyInput.csv" -WhatIf
[CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'Medium')]
param (
    [Parameter(Mandatory = $true, HelpMessage = "Path to the CSV file containing SiteURL, GroupName, and Users columns.")]
    [ValidateScript({ Test-Path -LiteralPath $_ -PathType Leaf })]
    [string]$CsvPath,

    [Parameter(HelpMessage = "Skip login and rely on an existing CLI session.")]
    [switch]$SkipLogin
)

begin {
    if (-not $SkipLogin) {
        Write-Verbose "Ensuring CLI for Microsoft 365 login session."
        m365 login --ensure --output json | Out-Null
    }

    Write-Host "Reading CSV input from '$CsvPath'" -ForegroundColor Cyan
    $Script:CsvData = Import-Csv -LiteralPath $CsvPath
    if (-not $Script:CsvData) {
        throw "The CSV file '$CsvPath' is empty or could not be parsed."
    }

    $Script:Results = [System.Collections.Generic.List[pscustomobject]]::new()
}

process {
    $rowIndex = 0
    foreach ($row in $Script:CsvData) {
        $rowIndex++
        Write-Progress -Activity "Processing groups" -Status "Item $rowIndex of $($Script:CsvData.Count)" -PercentComplete (($rowIndex / $Script:CsvData.Count) * 100)

        try {
            $groupJson = m365 spo group get --webUrl $row.SiteURL --name $row.GroupName --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve group '$($row.GroupName)': $groupJson"
            }

            $membersJson = m365 spo group member list --webUrl $row.SiteURL --groupName $row.GroupName --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to retrieve group members: $membersJson"
            }

            $existingMembers = @()
            if ($membersJson.Trim()) {
                $existingMembers = $membersJson | ConvertFrom-Json
            }
            $existingEmails = @($existingMembers | ForEach-Object { $_.Email })

            $targetEmails = $row.Users -split ';' | Where-Object { $_ -and $_.Trim() } | ForEach-Object { $_.Trim() }
            if (-not $targetEmails) {
                $Script:Results.Add([pscustomobject]@{
                    SiteUrl   = $row.SiteURL
                    GroupName = $row.GroupName
                    Email     = $null
                    Status    = 'Skipped'
                    Message   = 'No user specified in CSV row'
                })
                continue
            }

            foreach ($email in $targetEmails) {
                if ($existingEmails -contains $email) {
                    $Script:Results.Add([pscustomobject]@{
                        SiteUrl   = $row.SiteURL
                        GroupName = $row.GroupName
                        Email     = $email
                        Status    = 'Skipped'
                        Message   = 'Already a member'
                    })
                    continue
                }

                if (-not $PSCmdlet.ShouldProcess($email, "Add to $($row.GroupName)")) {
                    $Script:Results.Add([pscustomobject]@{
                        SiteUrl   = $row.SiteURL
                        GroupName = $row.GroupName
                        Email     = $email
                        Status    = 'WhatIf'
                        Message   = 'WhatIf: membership not changed'
                    })
                    continue
                }

                $addOutput = m365 spo group member add --webUrl $row.SiteURL --groupName $row.GroupName --emails $email --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    throw "Failed to add member '$email': $addOutput"
                }

                $Script:Results.Add([pscustomobject]@{
                    SiteUrl   = $row.SiteURL
                    GroupName = $row.GroupName
                    Email     = $email
                    Status    = 'Added'
                    Message   = 'User added successfully'
                })
            }
        }
        catch {
            $Script:Results.Add([pscustomobject]@{
                SiteUrl   = $row.SiteURL
                GroupName = $row.GroupName
                Email     = $row.Users
                Status    = 'Failed'
                Message   = $_.Exception.Message
            })
        }
    }
}

end {
    $summary = $Script:Results | Group-Object Status | Select-Object @{Name='Status';Expression={$_.Name}}, @{Name='Count';Expression={$_.Count}}
    Write-Host "========== Summary ==========" -ForegroundColor Cyan
    foreach ($item in $summary) {
        Write-Host ("{0,-10}: {1}" -f $item.Status, $item.Count)
    }
    Write-Host "=============================" -ForegroundColor Cyan

    $Script:Results | Sort-Object SiteUrl, GroupName, Email | Format-Table -AutoSize
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Chandani Prajapati |
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/about) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-add-bulk-users-to-groups" aria-hidden="true" />



# Get Site Collection invalid user accounts

## Summary

When you have an old site collection with a lot of users, it can be hard to keep track of which users are valid and which are not. This script will help you find all the invalid users in your site collection.

In this script I have checked for two things:
1. Users that are disabled in Azure AD
2. Users that are not in the User Profile Application

![Example Screenshot](assets/example.png)


# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
function Get-SpoInvalidUserAccounts {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site to scan")]
        [ValidateNotNullOrEmpty()]
        [string]
        $SiteUrl,

        [Parameter(HelpMessage = "Directory where the CSV report will be saved")]
        [ValidateNotNullOrEmpty()]
        [string]
        $ReportDirectory = "./reports",

        [Parameter(HelpMessage = "Skip user profile verification to speed up processing")]
        [switch]
        $SkipUserProfileCheck,

        [Parameter(HelpMessage = "Skip Microsoft Entra ID (Azure AD) account status validation")]
        [switch]
        $SkipAccountEnabledCheck
    )

    begin {
        Write-Host "Ensuring Microsoft 365 CLI session..." -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to ensure Microsoft 365 CLI login. CLI output: $loginOutput"
        }

        $resolvedDirectory = Resolve-Path -Path $ReportDirectory -ErrorAction SilentlyContinue
        if (-not $resolvedDirectory) {
            Write-Verbose "Creating report directory '$ReportDirectory'."
            $null = New-Item -ItemType Directory -Path $ReportDirectory -Force
            $resolvedDirectory = Resolve-Path -Path $ReportDirectory
        }

        $script:ReportPath = Join-Path -Path $resolvedDirectory.Path -ChildPath ("invalid-spo-users-{0}.csv" -f (Get-Date -Format 'yyyyMMdd-HHmmss'))

        $script:Summary = [ordered]@{
            TotalUsers          = 0
            InvalidUsers        = 0
            DisabledAccounts    = 0
            MissingProfiles     = 0
            ScriptFailures      = 0
        }
        $script:InvalidUsers = New-Object System.Collections.Generic.List[object]
    }

    process {
        $disabledAccounts = @()
        if (-not $SkipAccountEnabledCheck) {
            Write-Host "Retrieving Entra ID user account status..." -ForegroundColor Yellow
            $entraOutput = m365 entra user list --properties "userPrincipalName,mail,accountEnabled" --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to list Entra ID users. CLI output: $entraOutput"
            }

            try {
                $disabledAccounts = $entraOutput | ConvertFrom-Json -ErrorAction Stop | Where-Object {
                    $_.AccountEnabled -eq $false
                }
            }
            catch {
                throw "Unable to parse Entra ID response as JSON. $($_.Exception.Message)"
            }
        }

        Write-Host "Retrieving SharePoint users for $SiteUrl..." -ForegroundColor Yellow
        $spoUsersOutput = m365 spo user list --webUrl $SiteUrl --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list site users. CLI output: $spoUsersOutput"
        }

        try {
            $users = $spoUsersOutput | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse SharePoint user list response as JSON. $($_.Exception.Message)"
        }

        if (-not $users) {
            Write-Warning "The site returned no users."
            return
        }

        $script:Summary.TotalUsers = $users.Count
        Write-Host "Validating $($users.Count) user(s)..." -ForegroundColor Cyan

        foreach ($user in $users) {
            $principal = $user.UserPrincipalName
            if (-not $principal) {
                $principal = $user.Email
            }
            if (-not $principal) {
                $principal = $user.LoginName
            }

            $validationResults = [ordered]@{
                UserPrincipalName    = $principal
                LoginName            = $user.LoginName
                Email                = $user.Email
                Title                = $user.Title
                Reason               = @()
            }

            $isInvalid = $false

            if (-not $SkipAccountEnabledCheck -and $principal) {
                $disabledMatch = $disabledAccounts | Where-Object {
                    $_.UserPrincipalName -eq $principal -or $_.Mail -eq $principal
                }
                if ($disabledMatch) {
                    $validationResults.Reason += "Disabled in Entra ID"
                    $script:Summary.DisabledAccounts++
                    $isInvalid = $true
                }
            }

            if (-not $SkipUserProfileCheck -and $principal) {
                $profileOutput = m365 spo userprofile get --userName $principal --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    $validationResults.Reason += "Profile lookup failed"
                    $script:Summary.ScriptFailures++
                    $isInvalid = $true
                }
                else {
                    try {
                        $profile = $profileOutput | ConvertFrom-Json -ErrorAction Stop
                        if (-not $profile) {
                            $validationResults.Reason += "Missing SharePoint user profile"
                            $script:Summary.MissingProfiles++
                            $isInvalid = $true
                        }
                    }
                    catch {
                        $validationResults.Reason += "Profile JSON parse error"
                        $script:Summary.ScriptFailures++
                        $isInvalid = $true
                    }
                }
            }

            if ($isInvalid) {
                $script:Summary.InvalidUsers++
                $validationResults.Reason = $validationResults.Reason -join '; '
                $script:InvalidUsers.Add([pscustomobject]$validationResults)
            }
        }
    }

    end {
        if ($script:InvalidUsers.Count -gt 0) {
            if ($PSCmdlet.ShouldProcess($script:ReportPath, "Write invalid-user report")) {
                try {
                    $script:InvalidUsers | Export-Csv -Path $script:ReportPath -NoTypeInformation -Encoding UTF8
                    Write-Host "Invalid user report saved to $($script:ReportPath)." -ForegroundColor Green
                }
                catch {
                    $script:Summary.ScriptFailures++
                    Write-Error "Failed to write CSV report. $($_.Exception.Message)"
                }
            }
        }
        else {
            Write-Host "No invalid users found based on the selected checks." -ForegroundColor Green
        }

        Write-Host "----- Summary -----" -ForegroundColor Cyan
        Write-Host "Total users processed      : $($script:Summary.TotalUsers)"
        Write-Host "Invalid users identified   : $($script:Summary.InvalidUsers)"
        Write-Host "Disabled Entra accounts    : $($script:Summary.DisabledAccounts)"
        Write-Host "Missing SharePoint profiles: $($script:Summary.MissingProfiles)"
        Write-Host "Failures encountered       : $($script:Summary.ScriptFailures)"
        if ($script:InvalidUsers.Count -gt 0 -and (Test-Path -Path $script:ReportPath)) {
            Write-Host "Report path                : $($script:ReportPath)" -ForegroundColor Cyan
        }
    }
}

Get-SpoInvalidUserAccounts -SiteUrl "https://contoso.sharepoint.com/sites/workspaces" -ReportDirectory "./reports"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell



#extract all users from a site collection and check for validity
$SiteURL = "https://contoso.sharepoint.com/sites/workspaces"
if(-not $conn)
{
    $conn = Connect-PnPOnline -Url $SiteURL -Interactive -ReturnConnection
}

function Get-AllUsersFromUPA
{
    $allUPAusers = @()
    $UPAusers = Submit-PnPSearchQuery -Query "*" -SourceId "b09a7990-05ea-4af9-81ef-edfab16c4e31" -SelectProperties "Title,WorkEmail" -All -Connection $conn
    foreach($user in $UPAusers.ResultRows)
    {
        $allUPAusers += $user.LoginName
    }
    $allUPAusers
}

function Get-UserFromGraph 
{
    $disabledusersfromgraph = @()
    $result = Invoke-PnPGraphMethod -Url "users?`$select=displayName,mail, AccountEnabled" -Connection $conn

    $result.value.Count
    foreach($account in $result.value)
    {
        if($account.accountEnabled -eq $false)
        {
            $disabledusersfromgraph += $account.mail
        }
    }
    $disabledusersfromgraph
}

$disabledusersfromgraph = Get-UserFromGraph
$allUPAusers = Get-AllUsersFromUPA

$allSiteUsers = Get-PnPUser -Connection $conn
$validUsers = @()
$invalidUsers = @()
foreach($user in $allSiteUsers)
{
    try {
        $userObj = Get-PnPUser -Identity $user.LoginName -Connection $conn -ErrorAction Stop
        if($userObj.Email -in $disabledusersfromgraph)
        {
            Write-Host "User $($userObj.LoginName) is disabled in Azure AD"
            $invalidUsers += $user
        }
        else
        {
            $hit = $allUPAusers | Where-Object {$_ -eq $userObj.LoginName}
            if(-not $hit)
            {
                Write-Host "User $($userObj.LoginName) is not in the UPA"
                $invalidUsers += $user
            }
        }
        
        
    }
    catch {
        $invalidUsers += $user
    }
}
$invalidUsers | Export-Csv -Path "C:\temp\invalidusers.csv" -Delimiter "|" -Encoding utf8 -Force

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/get-spo-invalid-user-accounts" aria-hidden="true" />

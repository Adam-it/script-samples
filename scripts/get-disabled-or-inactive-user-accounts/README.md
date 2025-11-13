

# Get disabled or inactive user accounts

## Summary

In order to keep your tenant clean (Governance), you might want to ensure that disabled or inactive user accounts will be replaced where oppropriate (Think Owners of sites/groups, assignedto user on tasks/planner and so on). This script will help you find those accounts.

![Example Screenshot](assets/example.png)


# [PnP PowerShell](#tab/pnpps)

```powershell


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
function Get-UserFromSharePointSearch 
{
    $usersfromsearch = @()
    #How you tag an account as disabled varies from org to org, so you might need to change the below
    #in one tenant the account name was prefixed with ZZ_[Year of leaving]
    #in another tenant they had a custom property called EmployeeStatus, and sometimes a DateLeft property
    #SourceId "b09a7990-05ea-4af9-81ef-edfab16c4e31"  is the People source in SharePoint
    $results = Invoke-PnPSearchQuery -Query "*" -SourceId "b09a7990-05ea-4af9-81ef-edfab16c4e31" -All -Connection $conn    
    
    foreach($result in $results.ResultRows)
    {
        #you can replace this with whatever you use to tag an account as disabled
        if($result["SPS-HideFromAddressLists"] -eq $true)
        {
            $usersfromsearch += $result["WorkEmail"]
        }
    }
    $usersfromsearch
}
function Get-UserFromGraphThatHasntLoggedInResently($duration = 90) 
{
    $inactiveusersfromgraph = @()
    $authToken = Get-PnPGraphAccessToken -Connection $conn
    $uri = "https://graph.microsoft.com/v1.0/users"
    $Headers = @{
        "Authorization" = "Bearer $($authToken)"
        "Content-type"  = "application/json"
    }
    $response = Invoke-RestMethod -Headers $Headers -Uri $uri -Method GET
    foreach($user in $response.value)
    {
        # requires the AuditLog.Read.All permission
        $signinsUri = "https://graph.microsoft.com/v1.0/auditLogs/signIns?$top=1&$filter=userPrincipalName eq '$($user.userPrincipalName)')"
        $response = Invoke-RestMethod -Headers $Headers -Uri $signinsUri -Method GET
        
        if($response.value.Count -eq 0)
        {
            #no signin found
            $inactiveusersfromgraph += $user.userPrincipalName
        }
        else {
            if($response.value[0].createdDateTime -lt (Get-Date).AddDays(-$duration))
            {
                #user has not signed in for 90 days
                $inactiveusersfromgraph += $user.userPrincipalName
                
            }
        }
    }
    $inactiveusersfromgraph
}




$ClientId = "clientid"
$TenantName = "[domain].onmicrosoft.com"
$SharePointAdminSiteURL = "https://[domain]-admin.sharepoint.com/"
#connect to SharePoint using a certificate or similar
$conn = Connect-PnPOnline -Url $SharePointAdminSiteURL -ClientId $ClientId -Tenant $TenantName -CertificatePath "C:\Users\[you]\[CertName].pfx" -CertificatePassword (ConvertTo-SecureString -AsPlainText -Force "ThePassWord") -ReturnConnection

#get user data from graph and log those which are disabled
$userd1 = Get-UserFromGraph
$userd2 = Get-UserFromSharePointSearch
$users3 = Get-UserFromGraphThatHasntLoggedInResently

#output to csv file or use the data in some other way, like checking if the disabled users is a Owner of some site or group
$userd1 | Export-Csv -Path "C:\temp\disabledusers.csv" -NoTypeInformation 

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
# .\Get-DisabledOrInactiveUsers.ps1 -OutputCsv ".\reports\disabled-users.csv" -InactiveDays 90
[CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'Low')]
param (
    [Parameter(HelpMessage = "Number of days without sign-in to classify a user as inactive.")]
    [ValidateRange(30, 365)]
    [int]$InactiveDays = 90,

    [Parameter(Mandatory = $true, HelpMessage = "Path to the CSV file where the report will be exported.")]
    [ValidateNotNullOrEmpty()]
    [string]$OutputCsv
)

begin {
    Write-Verbose "Ensuring CLI for Microsoft 365 session."
    m365 login --ensure

    $Script:CutoffDate = (Get-Date).AddDays(-$InactiveDays)
    $Script:Report = [System.Collections.Generic.List[pscustomobject]]::new()

}

process {
    Write-Verbose "Retrieving disabled users from Microsoft Entra."
    $disabledUsersJson = m365 entra user list --filter "accountEnabled eq false" --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve disabled users. CLI output: $disabledUsersJson"
    }
    $disabledUsers = $disabledUsersJson | ConvertFrom-Json

    foreach ($user in $disabledUsers) {
        $Script:Report.Add([pscustomobject]@{
            UserPrincipalName = $user.userPrincipalName
            DisplayName       = $user.displayName
            LastSignIn        = $null
            Status            = 'Disabled'
        })
    }

    Write-Verbose "Retrieving last sign-in dates for active accounts."
    $signIns = @()
    $page = 0
    do {
        $page++
        Write-Verbose "Fetching sign-in page $page."
        $signInJson = m365 entra user signin list --output json --top 200 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve sign-in data. CLI output: $signInJson"
        }
        $pageResults = $signInJson | ConvertFrom-Json
        if ($pageResults) {
            $signIns += $pageResults
        }
        $more = $pageResults.Count -eq 200
    } while ($more -and $page -lt 10)

    $recentSignIns = @{}
    foreach ($record in $signIns) {
        $upn = $record.userPrincipalName
        if (-not $upn) { continue }
        $timestamp = Get-Date $record.createdDateTime
        if ($recentSignIns.ContainsKey($upn)) {
            if ($timestamp -gt $recentSignIns[$upn]) {
                $recentSignIns[$upn] = $timestamp
            }
        }
        else {
            $recentSignIns[$upn] = $timestamp
        }
    }

    Write-Verbose "Identifying inactive users (no sign-in within $InactiveDays days)."
    $activeUsersJson = m365 entra user list --filter "accountEnabled eq true" --select "id,displayName,userPrincipalName" --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve active users. CLI output: $activeUsersJson"
    }
    $activeUsers = $activeUsersJson | ConvertFrom-Json

    foreach ($user in $activeUsers) {
        $upn = $user.userPrincipalName
        $lastSignIn = $recentSignIns[$upn]
        if (-not $lastSignIn -or $lastSignIn -lt $Script:CutoffDate) {
            $Script:Report.Add([pscustomobject]@{
                UserPrincipalName = $upn
                DisplayName       = $user.displayName
                LastSignIn        = $lastSignIn
                Status            = 'Inactive'
            })
        }
    }
}

end {
    if (-not $Script:Report.Count) {
        Write-Host "No disabled or inactive accounts found." -ForegroundColor Green
        return
    }

    $destination = Resolve-Path -LiteralPath $OutputCsv -ErrorAction SilentlyContinue
    if (-not $destination) {
        $destination = (Resolve-Path -LiteralPath (Split-Path $OutputCsv -Parent -ErrorAction SilentlyContinue))
        if (-not $destination) {
            New-Item -ItemType Directory -Path (Split-Path $OutputCsv) -Force | Out-Null
        }
        $destination = $OutputCsv
    }

    if ($PSCmdlet.ShouldProcess($OutputCsv, 'Export disabled/inactive user report')) {
        $Script:Report | Sort-Object Status, UserPrincipalName | Export-Csv -Path $OutputCsv -NoTypeInformation
        Write-Host "Report exported to '$OutputCsv'." -ForegroundColor Green
    }
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/get-disabled-or-inactive-user-accounts" aria-hidden="true" />

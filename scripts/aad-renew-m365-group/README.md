

# Automate Renewal of Expiring M365 Groups or or Microsoft Teams teams

## Summary

It is a good practice to set lifecycle expiration policy to control sprawl. However that means that the group will get automatically deleted after they expire. The Teams/M365 groups owners will get email notifications to renew within a certain timeframe, however if the owners missed the renewal notifications for different reasons, it may lead to accidental data loss. The script can help identify M365 groups nearing expiration to renew them. This sample is available in both PnP PowerShell and CLI for Microsoft 365.

![Example Screenshot](assets/example.png)

# [PnP PowerShell](#tab/pnpps)

```powershell
param (
    [Parameter(Mandatory = $true)]
    [string] $domain
)

$adminSiteURL = "https://$domain-Admin.SharePoint.com"
$dateTime = "_{0:MM_dd_yy}_{0:HH_mm_ss}" -f (Get-Date)
$invocation = (Get-Variable MyInvocation).Value
$directorypath = Split-Path $invocation.MyCommand.Path
$fileName = "m365_group_expire_reset" + $dateTime + ".csv"
$outputPath = $directorypath + "\"+ $fileName

if (-not (Test-Path $outputPath)) {
    New-Item -ItemType File -Path $outputPath
}
Connect-PnPOnline -Url $adminSiteURL -Interactive -WarningAction SilentlyContinue


Get-PnPMicrosoft365ExpiringGroup  | ForEach-Object {
    $group = $_
    Reset-PnPMicrosoft365GroupExpiration -Identity $group.Id
    $group = Get-PnPMicrosoft365Group -Identity $group.Id
    $group | Select-Object id, RenewedDateTime,DisplayName|Export-Csv -Path $outputPath -NoTypeInformation -Append

}

Disconnect-PnPOnline
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(HelpMessage = "Groups expiring within this many days will be renewed")]
    [ValidateRange(1, 365)]
    [int]$DaysBeforeExpiration = 30,
    
    [Parameter(HelpMessage = "Path where the CSV file will be saved")]
    [string]$OutputPath = "m365_group_renewals_$(Get-Date -Format 'MM-dd-yyyy-HH-mm-ss').csv",
    
    [Parameter(HelpMessage = "Export results to CSV file")]
    [switch]$ExportToCsv
)

begin {
    Write-Verbose "Starting M365 Group renewal script"
    
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
        GroupsFound = 0
        GroupsExpiring = 0
        GroupsRenewed = 0
        Failures = 0
    }
    
    $script:Report = [System.Collections.ArrayList]::new()
}

process {
    Write-Verbose "Retrieving all Microsoft 365 Groups"
    $groupsJson = m365 entra m365group list --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve groups. CLI output: $groupsJson"
    }
    
    $allGroups = @($groupsJson | ConvertFrom-Json)
    $script:Summary.GroupsFound = $allGroups.Count
    Write-Verbose "Found $($allGroups.Count) total group(s)"
    
    $currentDate = Get-Date
    $expiringGroups = $allGroups | Where-Object {
        if ($_.expirationDateTime) {
            $daysLeft = ([DateTime]$_.expirationDateTime - $currentDate).Days
            $daysLeft -le $DaysBeforeExpiration -and $daysLeft -ge 0
        }
    }
    
    $script:Summary.GroupsExpiring = $expiringGroups.Count
    Write-Verbose "Found $($expiringGroups.Count) group(s) expiring within $DaysBeforeExpiration days"
    
    if ($expiringGroups.Count -eq 0) {
        Write-Warning "No groups found expiring within $DaysBeforeExpiration days"
        return
    }
    
    foreach ($group in $expiringGroups) {
        $daysUntilExpiration = ([DateTime]$group.expirationDateTime - $currentDate).Days
        Write-Verbose "Processing group: $($group.displayName) (expires in $daysUntilExpiration days)"
        
        try {
            if ($PSCmdlet.ShouldProcess($group.displayName, "Renew group expiration")) {
                Write-Verbose "  Renewing group expiration"
                m365 entra m365group renew --id $($group.id) 2>&1 | Out-Null
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to renew group '$($group.displayName)'"
                    $script:Summary.Failures++
                    [void]$script:Report.Add([PSCustomObject]@{
                        GroupId = $group.id
                        DisplayName = $group.displayName ?? "(No Display Name)"
                        Mail = $group.mail
                        Description = $group.description
                        ExpirationDate = $group.expirationDateTime
                        DaysUntilExpiration = $daysUntilExpiration
                        RenewedDateTime = $null
                        Status = "Failed to renew"
                        ErrorMessage = "CLI command failed"
                    })
                    continue
                }
                
                Write-Verbose "  Verifying renewal"
                $updatedJson = m365 entra m365group get --id $($group.id) --output json 2>&1
                if ($LASTEXITCODE -eq 0) {
                    $updatedGroup = $updatedJson | ConvertFrom-Json
                    $script:Summary.GroupsRenewed++
                    
                    [void]$script:Report.Add([PSCustomObject]@{
                        GroupId = $updatedGroup.id
                        DisplayName = $updatedGroup.displayName ?? "(No Display Name)"
                        Mail = $updatedGroup.mail
                        Description = $updatedGroup.description
                        ExpirationDate = $updatedGroup.expirationDateTime
                        DaysUntilExpiration = $daysUntilExpiration
                        RenewedDateTime = $updatedGroup.renewedDateTime
                        Status = "Renewed"
                        ErrorMessage = ""
                    })
                    Write-Verbose "  Group renewed successfully"
                } else {
                    Write-Warning "Group renewed but failed to verify: $updatedJson"
                    $script:Summary.GroupsRenewed++
                    [void]$script:Report.Add([PSCustomObject]@{
                        GroupId = $group.id
                        DisplayName = $group.displayName ?? "(No Display Name)"
                        Mail = $group.mail
                        Description = $group.description
                        ExpirationDate = $group.expirationDateTime
                        DaysUntilExpiration = $daysUntilExpiration
                        RenewedDateTime = "Unknown"
                        Status = "Renewed (unverified)"
                        ErrorMessage = "Failed to verify renewal"
                    })
                }
            } else {
                Write-Verbose "  WhatIf: Would renew group"
            }
            
        } catch {
            Write-Warning "Error processing group '$($group.displayName)': $($_.Exception.Message)"
            $script:Summary.Failures++
            [void]$script:Report.Add([PSCustomObject]@{
                GroupId = $group.id
                DisplayName = $group.displayName ?? "(No Display Name)"
                Mail = $group.mail
                Description = $group.description
                ExpirationDate = $group.expirationDateTime
                DaysUntilExpiration = $daysUntilExpiration
                RenewedDateTime = $null
                Status = "Exception"
                ErrorMessage = $_.Exception.Message
            })
        }
    }
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "M365 Group Renewal Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Total groups found:   $($script:Summary.GroupsFound)" -ForegroundColor White
    Write-Host "Groups expiring:      $($script:Summary.GroupsExpiring)" -ForegroundColor Yellow
    Write-Host "Groups renewed:       $($script:Summary.GroupsRenewed)" -ForegroundColor Green
    Write-Host "Failures:             $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Green' })
    Write-Host "========================================`n" -ForegroundColor Cyan
    
    if ($ExportToCsv -and $script:Report.Count -gt 0) {
        if ($PSCmdlet.ShouldProcess($OutputPath, "Export results to CSV")) {
            $script:Report | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
            Write-Host "Results exported to: $OutputPath" -ForegroundColor Green
        }
    } elseif ($script:Report.Count -gt 0) {
        Write-Host "Expiring Groups Renewal Results:" -ForegroundColor Yellow
        $script:Report | Format-Table -AutoSize
    }
    
    Write-Verbose "Script execution completed"
}
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Source Credit

Sample first appeared on [Automate Renewal of Expiring M365 Groups Using PowerShell](https://reshmeeauckloo.com/posts/powershell-renew-expiring-m365-group/)


## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011) |
| Adam Wójcik [@Adam-it](https://github.com/Adam-it)|


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/aad-renew-m365-group" aria-hidden="true" />

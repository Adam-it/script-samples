# Export & Compare Conditional-Access Policies (drift-detect)

## Summary

This script automates the export and comparison of Conditional Access Policies within an Azure AD tenant to detect configuration drift. It helps monitor changes to these critical security controls over time, ensuring they remain compliant with your organization's zero-trust baselines.

The script performs the following operations:
1. Authenticates to Microsoft Graph API
2. Exports all Conditional Access Policies to JSON format
3. Saves each policy with timestamps to establish a version history
4. Compares current policies to the previous export
5. Generates alerts for any detected changes
6. Optionally sends notifications via email or Microsoft Teams

![Conditional Access Policy Drift Detection](assets/example.png)

## Prerequisites

- Microsoft Graph PowerShell SDK modules installed
- Permissions to read Conditional Access Policies (Application or delegated)
- Required permissions:
  - `Policy.Read.All` for reading Conditional Access Policies
  - `Mail.Send` (optional for email notifications)
  - Teams webhook (optional for Teams notifications)

## Implementation

This solution provides a scheduled monitoring approach for Conditional Access Policies in your tenant. By tracking policy changes, your security team can quickly identify unexpected alterations, ensuring there's no drift from your security baselines.

# [Microsoft Graph PowerShell](#tab/graphps)

```powershell
# Conditional Access Policy Export and Drift Detection Script
# Script exports all Conditional Access Policies from Azure AD, saves them to JSON files,
# and compares them with previous exports to detect any changes (drift)

#-------------------------------------------------------------
# Module Management
#-------------------------------------------------------------
# Required modules
$requiredModules = @(
    "Microsoft.Graph.Authentication", 
    "Microsoft.Graph.Identity.SignIns"
)

# Check and install required modules
foreach ($module in $requiredModules) {
    # Check if module is installed
    if (-not (Get-Module -ListAvailable -Name $module)) {
        Write-Host "Module $module is not installed. Installing..."
        try {
            Install-Module -Name $module -Force -AllowClobber -Scope CurrentUser
            Write-Host "Module $module installed successfully" -ForegroundColor Green
        }
        catch {
            Write-Host "Failed to install module $module. Error: $_" -ForegroundColor Red
            exit 1
        }
    }
    
    # Import the module
    try {
        Import-Module -Name $module -ErrorAction Stop
        Write-Host "Module $module imported successfully" -ForegroundColor Green
    }
    catch {
        Write-Host "Failed to import module $module. Error: $_" -ForegroundColor Red
        exit 1
    }
}

#-------------------------------------------------------------
# Configuration
#-------------------------------------------------------------
$ConfigPath = "$PSScriptRoot\Config"
$HistoryPath = "$PSScriptRoot\History"
$CurrentExportPath = "$PSScriptRoot\Current"
$LogPath = "$PSScriptRoot\Logs"
$ComparisonReportPath = "$PSScriptRoot\Reports"
$SendEmail = $false
$SendTeamsNotification = $true

# Email settings (if $SendEmail is $true)
$EmailFrom = "[your email address]"
$EmailTo = "[recipient email address]"
$SmtpServer = "smtp.office365.com"

# Teams webhook URL (if $SendTeamsNotification is $true)
$TeamsWebhookUrl = "https://[your tenant].webhook.office.com/webhookb2/"

# Create required directories if they don't exist
$Directories = @($ConfigPath, $HistoryPath, $CurrentExportPath, $LogPath, $ComparisonReportPath)
foreach ($Dir in $Directories) {
    if (!(Test-Path -Path $Dir)) {
        New-Item -ItemType Directory -Path $Dir -Force | Out-Null
    }
}

#-------------------------------------------------------------
# Functions
#-------------------------------------------------------------
function Write-Log {
    param (
        [string]$Message,
        [string]$Level = "INFO"
    )
    
    $Timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $LogEntry = "[$Timestamp] [$Level] $Message"
    
    # Write to console
    switch ($Level) {
        "ERROR" { Write-Host $LogEntry -ForegroundColor Red }
        "WARNING" { Write-Host $LogEntry -ForegroundColor Yellow }
        "SUCCESS" { Write-Host $LogEntry -ForegroundColor Green }
        default { Write-Host $LogEntry }
    }
    
    # Write to log file
    $LogFile = Join-Path -Path $LogPath -ChildPath "CA-Drift-$(Get-Date -Format 'yyyy-MM-dd').log"
    Add-Content -Path $LogFile -Value $LogEntry
}

function Send-EmailAlert {
    param (
        [string]$Subject,
        [string]$Body
    )
    
    try {
        Send-MailMessage -From $EmailFrom -To $EmailTo -Subject $Subject -Body $Body -BodyAsHtml -SmtpServer $SmtpServer -UseSsl -Port 587 -Credential (Get-Credential -Message "Enter email credentials")
        Write-Log "Email notification sent successfully" -Level "SUCCESS"
    }
    catch {
        Write-Log "Failed to send email notification: $_" -Level "ERROR"
    }
}

function Send-TeamsAlert {
    param (
        [string]$Title,
        [string]$Message,
        [string]$Color = "#FF0000" # Red
    )
    
    try {
        $JSON = @{
            "@type" = "MessageCard"
            "@context" = "http://schema.org/extensions"
            "summary" = $Title
            "themeColor" = $Color
            "sections" = @(
                @{
                    "activityTitle" = $Title
                    "activitySubtitle" = "Generated on $(Get-Date -Format 'yyyy-MM-dd HH:mm')"
                    "text" = $Message
                }
            )
        } | ConvertTo-Json -Depth 4
        
        Invoke-RestMethod -Uri $TeamsWebhookUrl -Method Post -Body $JSON -ContentType "application/json"
        Write-Log "Teams notification sent successfully" -Level "SUCCESS"
    }
    catch {
        Write-Log "Failed to send Teams notification: $_" -Level "ERROR"
    }
}

function Compare-Policies {
    param (
        [string]$CurrentPolicyPath,
        [string]$PreviousPolicyPath,
        [string]$PolicyName
    )
    
    try {
        $CurrentPolicy = Get-Content -Path $CurrentPolicyPath | ConvertFrom-Json
        $PreviousPolicy = Get-Content -Path $PreviousPolicyPath | ConvertFrom-Json
        
        # Compare policies using Compare-Object
        $Comparison = Compare-Object -ReferenceObject ($PreviousPolicy | ConvertTo-Json -Depth 10) -DifferenceObject ($CurrentPolicy | ConvertTo-Json -Depth 10)
        
        if ($Comparison) {
            # Policies are different
            Write-Log "DRIFT DETECTED: Changes found in policy '$PolicyName'" -Level "WARNING"
            
            # Generate detailed comparison report using a more detailed approach
            $Report = @{
                PolicyName = $PolicyName
                ChangeDetected = $true
                ChangeTimestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                PreviousVersion = $PreviousPolicy
                CurrentVersion = $CurrentPolicy
                Changes = @()
            }
            
            # Use PowerShell's Compare-Object to do a property-by-property comparison
            # This is a simplified approach - in reality you would recurse through nested properties
            foreach ($Property in $CurrentPolicy.PSObject.Properties.Name) {
                if ($CurrentPolicy.$Property -ne $PreviousPolicy.$Property) {
                    $Report.Changes += @{
                        Property = $Property
                        PreviousValue = $PreviousPolicy.$Property
                        CurrentValue = $CurrentPolicy.$Property
                    }
                }
            }
            
            # Save the comparison report
            $ReportFilePath = Join-Path -Path $ComparisonReportPath -ChildPath "$PolicyName-Changes-$(Get-Date -Format 'yyyy-MM-dd-HHmmss').json"
            $Report | ConvertTo-Json -Depth 10 | Out-File -FilePath $ReportFilePath
            
            return $Report
        }
        else {
            # No changes
            Write-Log "No changes detected in policy '$PolicyName'" -Level "INFO"
            return $null
        }
    }
    catch {
        Write-Log "Error comparing policies for '$PolicyName': $_" -Level "ERROR"
        return $null
    }
}

#-------------------------------------------------------------
# Main Script
#-------------------------------------------------------------
Write-Log "Starting Conditional Access Policy export and drift detection" -Level "INFO"

try {
    # Connect to Microsoft Graph
    Write-Log "Connecting to Microsoft Graph..."
    Connect-MgGraph -Scopes "Policy.Read.All" -NoWelcome
    
    # Get current date/time for timestamping
    $Timestamp = Get-Date -Format "yyyy-MM-dd-HHmmss"
    
    # Create a directory for this export in the history
    $CurrentExportDir = Join-Path -Path $HistoryPath -ChildPath $Timestamp
    New-Item -ItemType Directory -Path $CurrentExportDir -Force | Out-Null
    
    # Get all Conditional Access Policies
    Write-Log "Retrieving Conditional Access Policies..."
    $Policies = Invoke-MgGraphRequest -Uri 'https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies' -Method GET
    
    if ($Policies.value.Count -eq 0) {
        Write-Log "No Conditional Access Policies found in the tenant" -Level "WARNING"
    }
    else {
        Write-Log "Found $($Policies.value.Count) Conditional Access Policies" -Level "SUCCESS"
        
        $ChangesDetected = $false
        $ChangedPolicies = @()
        
        # Process each policy
        foreach ($Policy in $Policies.value) {
            # Clean policy name for file naming (remove invalid chars)
            $SafePolicyName = $Policy.displayName -replace '[\\/*?:"<>|]', '_'
            
            # Export current policy to the Current folder
            $CurrentPolicyPath = Join-Path -Path $CurrentExportPath -ChildPath "$SafePolicyName.json"
            $Policy | ConvertTo-Json -Depth 10 | Out-File -FilePath $CurrentPolicyPath
            
            # Save to the history folder
            $HistoryPolicyPath = Join-Path -Path $CurrentExportDir -ChildPath "$SafePolicyName.json"
            $Policy | ConvertTo-Json -Depth 10 | Out-File -FilePath $HistoryPolicyPath
            
            Write-Log "Exported policy: $($Policy.displayName)" -Level "INFO"
            
            # Find the most recent previous version of this policy (if any)
            $PreviousVersions = Get-ChildItem -Path $HistoryPath -Recurse -Filter "$SafePolicyName.json" | 
                                Where-Object { $_.FullName -ne $HistoryPolicyPath } | 
                                Sort-Object LastWriteTime -Descending
            
            if ($PreviousVersions.Count -gt 0) {
                $PreviousPolicyPath = $PreviousVersions[0].FullName
                
                # Compare with previous version
                $ComparisonResult = Compare-Policies -CurrentPolicyPath $CurrentPolicyPath -PreviousPolicyPath $PreviousPolicyPath -PolicyName $Policy.displayName
                
                if ($ComparisonResult) {
                    $ChangesDetected = $true
                    $ChangedPolicies += $ComparisonResult
                }
            }
            else {
                Write-Log "No previous version found for policy '$($Policy.displayName)' - this appears to be new" -Level "INFO"
            }
        }
        
        # Handle notifications if changes were detected
        if ($ChangesDetected) {
            Write-Log "Changes detected in Conditional Access Policies!" -Level "WARNING"
            
            # Prepare notification content
            $NotificationTitle = "⚠️ Conditional Access Policy Changes Detected"
            $NotificationBody = @"
<h2>Conditional Access Policy Drift Detection</h2>
<p>Changes were detected in the following policies:</p>
<ul>
$($ChangedPolicies | ForEach-Object { "<li><strong>$($_.PolicyName)</strong> - Changed on $($_.ChangeTimestamp)</li>" })
</ul>
<p>Please review the detailed comparison reports in the following location:</p>
<p><code>$ComparisonReportPath</code></p>
"@
            
            # Send notifications if configured
            if ($SendEmail) {
                Send-EmailAlert -Subject $NotificationTitle -Body $NotificationBody
            }
            
            if ($SendTeamsNotification) {
                Send-TeamsAlert -Title $NotificationTitle -Message $NotificationBody
            }
        }
        else {
            Write-Log "No changes detected in any Conditional Access Policies" -Level "SUCCESS"
        }
    }
    
    # Disconnect from Microsoft Graph
    Disconnect-MgGraph | Out-Null
    Write-Log "Script completed successfully" -Level "SUCCESS"
}
catch {
    Write-Log "Error executing script: $_" -Level "ERROR"
    
    # Try to disconnect if connected
    try {
        Disconnect-MgGraph | Out-Null
    }
    catch {
        # Ignore disconnect errors
    }
}

```
[!INCLUDE [More about Microsoft Graph PowerShell SDK](../../docfx/includes/MORE-GRAPHSDK.md)]
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Invoke-CaPolicyDriftDetection {
    <#
    .SYNOPSIS
    Exports and compares Entra Conditional Access policies using CLI for Microsoft 365.

    .DESCRIPTION
    Authenticates via CLI, exports current policies, stores timestamped snapshots, compares against the
    previous export, and writes drift reports. Optional notifications can be layered on top of the JSON output.

    .PARAMETER ExportPath
    Root directory to store exports, history, logs, and reports. Defaults to ./CA-Policy-Audit.

    .PARAMETER Force
    Skip confirmation prompts when creating directories or overwriting files.

    .EXAMPLE
    Invoke-CaPolicyDriftDetection -ExportPath ./CA-Audit -Verbose
    #>
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter()]
        [ValidateNotNullOrEmpty()]
        [string]$ExportPath = './CA-Policy-Audit',

        [Parameter()]
        [switch]$Force
    )

    begin {
        Write-Host 'Ensuring CLI authentication.' -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "CLI login failed. Output: $loginOutput"
        }

        $script:Root = Resolve-Path -Path $ExportPath -ErrorAction SilentlyContinue
        if (-not $script:Root) {
            if (-not ($Force -or $PSCmdlet.ShouldProcess($ExportPath, 'Create export directory'))) {
                throw "ExportPath '$ExportPath' does not exist."
            }
            $script:Root = New-Item -ItemType Directory -Path $ExportPath -Force | Resolve-Path
        }

        $script:Paths = [ordered]@{
            Current  = Join-Path $script:Root '\Current'
            History  = Join-Path $script:Root '\History'
            Reports  = Join-Path $script:Root '\Reports'
            Logs     = Join-Path $script:Root '\Logs'
        }

        foreach ($path in $script:Paths.GetEnumerator()) {
            if (-not (Test-Path -Path $path.Value -PathType Container)) {
                New-Item -ItemType Directory -Path $path.Value -Force | Out-Null
            }
        }

        $script:Timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
        $script:CurrentSnapshot = Join-Path $script:Paths.Current "Policies-$($script:Timestamp).json"
        $script:LogFile = Join-Path $script:Paths.Logs "Run-$($script:Timestamp).log"

        $script:HistoryFiles = Get-ChildItem -Path $script:Paths.History -Filter 'Policies-*.json' | Sort-Object LastWriteTime -Descending
        $script:PreviousSnapshot = $script:HistoryFiles | Select-Object -First 1

        $script:Summary = [ordered]@{
            PoliciesExported = 0
            PoliciesChanged  = 0
        }
    }

    process {
        Write-Host 'Exporting current Conditional Access policies.' -ForegroundColor Cyan
        $policiesJson = m365 entra policy conditionalaccess list --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Unable to list conditional access policies. CLI output: $policiesJson"
        }

        try {
            $policies = $policiesJson | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Unable to parse policies. $($_.Exception.Message)"
        }

        if (-not $policies) {
            Write-Warning 'No conditional access policies returned.'
            return
        }

        $script:Summary.PoliciesExported = $policies.Count
        $policies | ConvertTo-Json -Depth 10 | Set-Content -Path $script:CurrentSnapshot -Encoding UTF8

        $historyCopy = Join-Path $script:Paths.History (Split-Path -Leaf $script:CurrentSnapshot)
        Copy-Item -Path $script:CurrentSnapshot -Destination $historyCopy -Force

        if (-not $script:PreviousSnapshot) {
            Write-Host 'Initial snapshot stored; no previous version to compare.' -ForegroundColor Yellow
            return
        }

        Write-Host "Comparing with previous snapshot '$(Split-Path -Leaf $script:PreviousSnapshot.FullName)'." -ForegroundColor Cyan
        $previousPolicies = Get-Content -Path $script:PreviousSnapshot.FullName | ConvertFrom-Json

        $diff = Compare-Object -ReferenceObject ($previousPolicies | ConvertTo-Json -Depth 10) `
                              -DifferenceObject ($policies | ConvertTo-Json -Depth 10)

        if ($diff) {
            $script:Summary.PoliciesChanged = 1
            $reportPath = Join-Path $script:Paths.Reports "Diff-$($script:Timestamp).json"
            $report = [ordered]@{
                Timestamp        = Get-Date
                PreviousSnapshot = $script:PreviousSnapshot.FullName
                CurrentSnapshot  = $script:CurrentSnapshot
                Policies         = @()
            }

            foreach ($policy in $policies) {
                $previous = $previousPolicies | Where-Object { $_.Id -eq $policy.Id }
                if (-not $previous) {
                    $report.Policies += [ordered]@{
                        PolicyId    = $policy.Id
                        DisplayName = $policy.DisplayName
                        ChangeType  = 'Added'
                        Current     = $policy
                        Previous    = $null
                    }
                    continue
                }

                $policyDiff = Compare-Object -ReferenceObject ($previous | ConvertTo-Json -Depth 10) -DifferenceObject ($policy | ConvertTo-Json -Depth 10)
                if ($policyDiff) {
                    $report.Policies += [ordered]@{
                        PolicyId    = $policy.Id
                        DisplayName = $policy.DisplayName
                        ChangeType  = 'Modified'
                        Current     = $policy
                        Previous    = $previous
                    }
                }
            }

            foreach ($previous in $previousPolicies) {
                if (-not ($policies | Where-Object { $_.Id -eq $previous.Id })) {
                    $report.Policies += [ordered]@{
                        PolicyId    = $previous.Id
                        DisplayName = $previous.DisplayName
                        ChangeType  = 'Removed'
                        Current     = $null
                        Previous    = $previous
                    }
                }
            }

            $report | ConvertTo-Json -Depth 10 | Set-Content -Path $reportPath -Encoding UTF8
            Write-Host "Drift detected. Report saved to '$reportPath'." -ForegroundColor Yellow
        }
        else {
            Write-Host 'No differences detected between current and previous policies.' -ForegroundColor Green
        }
    }

    end {
        $log = [ordered]@{
            Timestamp        = Get-Date
            PoliciesExported = $script:Summary.PoliciesExported
            PoliciesChanged  = $script:Summary.PoliciesChanged
            CurrentSnapshot  = $script:CurrentSnapshot
            PreviousSnapshot = $script:PreviousSnapshot?.FullName
        }
        $log | ConvertTo-Json -Depth 5 | Set-Content -Path $script:LogFile -Encoding UTF8

        Write-Host "Policies exported: $($script:Summary.PoliciesExported)" -ForegroundColor Cyan
        Write-Host "Policies changed: $($script:Summary.PoliciesChanged)" -ForegroundColor Cyan
        Write-Host "Current snapshot: $script:CurrentSnapshot" -ForegroundColor Cyan
    }
}

# example usage
Invoke-CaPolicyDriftDetection -ExportPath ./CA-Policy-Audit -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***


## Contributors

| Author(s) |
|-----------|
| [Valeras Narbutas](https://github.com/ValerasNarbutas) |
| Adam Wójcik |

## Version history

| Version | Date | Comments |
|---------|------|----------|
| 1.0 | May 25, 2025 | Initial release |

## Key learning points

1. Using Microsoft Graph PowerShell to access and export Conditional Access Policies
2. Implementing versioning for configuration tracking
3. Detecting changes (drift) between policy versions
4. Creating a notification system for security policy changes

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/aad-export-compare-conditional-access-policies" aria-hidden="true" />

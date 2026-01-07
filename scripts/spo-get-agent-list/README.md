

# Get list of SharePoint Agents with PowerShell

## Summary

This script will get all the SharePoint agents from the SharePoint document libraries and export the data to a CSV file. Data includes agent name, description, instructions, welcome message, conversation starters, capabilities, referenced URLs, site IDs, web IDs, list IDs, and unique IDs.

![Example Screenshot](assets/preview.png)

**Minimum Steps To Success**

* Update SharePoint url.
* Update file location.
* Run the script with your credentials.

### Prerequisites

Account or Entra app that runs the script needs access to all of the locations where the files, folders, libraries or sites to be used in the agent are located.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding()]
param (
    [Parameter(Mandatory, HelpMessage = "The SharePoint site URL to search for agent files")]
    [string]$SiteUrl,

    [Parameter(HelpMessage = "The output path for the CSV file (defaults to current directory)")]
    [string]$OutputPath
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = "GetSharePointAgents_$timestamp.log"
    Start-Transcript -Path $transcriptPath

    if (-not $OutputPath) {
        $OutputPath = (Get-Location).Path
    }
    
    if ($PSBoundParameters.ContainsKey('OutputPath') -and -not (Test-Path -Path $OutputPath)) {
        throw "Output path does not exist: $OutputPath"
    }

    $csvPath = Join-Path -Path $OutputPath -ChildPath "SharePointAgents_$timestamp.csv"

    Write-Host "[1/4] Ensuring CLI for Microsoft 365 login..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Exit code: $LASTEXITCODE"
    }
    Write-Host "Successfully authenticated." -ForegroundColor Green

    $script:AgentCollection = @()
}

process {
    try {
        Write-Host "`n[2/4] Retrieving document libraries from site..." -ForegroundColor Cyan
        $listsJson = m365 spo list list --webUrl $SiteUrl --filter "BaseTemplate eq 101 and Hidden eq false" --query "[?!(contains(['Form Templates','Style Library','Site Pages'], Title))]" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve document libraries. CLI: $listsJson"
        }

        $libraries = @($listsJson | ConvertFrom-Json)
        Write-Host "Found $($libraries.Count) document library(ies) to search (excluding system libraries)." -ForegroundColor Green

        if ($libraries.Count -eq 0) {
            Write-Host "No user document libraries found. Exiting." -ForegroundColor Yellow
            return
        }

        Write-Host "`n[3/4] Searching for .agent files..." -ForegroundColor Cyan
        $totalAgentsFound = 0

        foreach ($library in $libraries) {
            Write-Verbose "Searching in library: $($library.Title)"

            try {
                $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listId $library.Id --fields "FileRef,FileLeafRef" --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve items from '$($library.Title)'. CLI: $itemsJson"
                    continue
                }

                $items = @($itemsJson | ConvertFrom-Json)
                $agentFiles = $items | Where-Object { $_.FileLeafRef -like "*.agent" }

                if ($agentFiles.Count -gt 0) {
                    Write-Host "  Found $($agentFiles.Count) agent file(s) in '$($library.Title)'" -ForegroundColor White
                    $totalAgentsFound += $agentFiles.Count
                }

                foreach ($agentFile in $agentFiles) {
                    $fileRef = $agentFile.FileRef
                    $fileName = $agentFile.FileLeafRef

                    Write-Verbose "Processing agent file: $fileName"

                    try {
                        $fileContentJson = m365 spo file get --webUrl $SiteUrl --url $fileRef --asString 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            Write-Warning "Failed to download '$fileName'. CLI: $fileContentJson"
                            continue
                        }

                        $agentData = $fileContentJson | ConvertFrom-Json

                        $agentRecord = [PSCustomObject]@{
                            FileName              = $fileName
                            FilePath              = $fileRef
                            SchemaVersion         = $agentData.schemaVersion
                            CopilotName           = $agentData.customCopilotConfig.gptDefinition.name
                            Description           = $agentData.customCopilotConfig.gptDefinition.description
                            Instructions          = $agentData.customCopilotConfig.gptDefinition.instructions
                            WelcomeMessage        = $agentData.customCopilotConfig.conversationStarters.welcomeMessage.text
                            ConversationStarters  = ($agentData.customCopilotConfig.conversationStarters.conversationStarterList.text -join "; ")
                            Capabilities          = ($agentData.customCopilotConfig.gptDefinition.capabilities.name -join "; ")
                            ReferencedURLs        = ($agentData.customCopilotConfig.gptDefinition.capabilities.items_by_url.url -join "; ")
                            SiteIDs               = ($agentData.customCopilotConfig.gptDefinition.capabilities.items_by_url.site_id -join "; ")
                            WebIDs                = ($agentData.customCopilotConfig.gptDefinition.capabilities.items_by_url.web_id -join "; ")
                            ListIDs               = ($agentData.customCopilotConfig.gptDefinition.capabilities.items_by_url.list_id -join "; ")
                            UniqueIDs             = ($agentData.customCopilotConfig.gptDefinition.capabilities.items_by_url.unique_id -join "; ")
                        }

                        $script:AgentCollection += $agentRecord
                        Write-Host "  ✓ Processed: $fileName" -ForegroundColor Green

                    } catch {
                        Write-Warning "Error processing agent file '$fileName': $_"
                        continue
                    }
                }
            } catch {
                Write-Warning "Error searching in library '$($library.Title)': $_"
                continue
            }
        }
    } catch {
        Write-Error "Critical error during execution: $_"
        throw
    }
}

end {
    Write-Host "`n[4/4] Exporting results..." -ForegroundColor Cyan

    if ($script:AgentCollection.Count -gt 0) {
        $script:AgentCollection | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8
        Write-Host "`n========================================" -ForegroundColor Green
        Write-Host "Export Complete!" -ForegroundColor Green
        Write-Host "========================================" -ForegroundColor Green
        Write-Host "Total agents found: " -NoNewline
        Write-Host $script:AgentCollection.Count -ForegroundColor Green
        Write-Host "CSV file location: " -NoNewline
        Write-Host $csvPath -ForegroundColor Green
        Write-Host "Transcript log: " -NoNewline
        Write-Host $transcriptPath -ForegroundColor Green
    } else {
        Write-Host "`nNo SharePoint Agent files found in site." -ForegroundColor Yellow
    }

    Stop-Transcript
}

# Example 1: Export all agents from a site
# .\Get-SharePointAgentList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Company311"

# Example 2: Export to custom directory
# .\Get-SharePointAgentList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Company311" -OutputPath "C:\Reports"

# Example 3: With verbose output to see detailed processing
# .\Get-SharePointAgentList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Company311" -Verbose

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

# [PnP PowerShell](#tab/pnpps)

```powershell

$url = "https://[TenantName].sharepoint.com/sites/[SiteName]"
$fileLocation = "C:\Projects\temp\SharePointAgentFiles.csv"

# Connect-PnPOnline -Url $url -Interactive
Connect-PnPOnline -Url $url -UseWebLogin 

# Get all document libraries excluding system libraries
$documentLibraries = Get-PnPList | Where-Object {
    $_.BaseTemplate -eq 101 -and
    $_.Hidden -eq $false -and
    $_.Title -notmatch "Form Templates|Style Library|Site Pages"
}

$allAgentFiles = @()

foreach ($library in $documentLibraries) {

    # Extract all agent files from the library, basically all files with .agent extension. FieldValues contains all the metadata of the file.
    $files = Get-PnPListItem -List $library.Title | Where-Object { $_["FileLeafRef"] -like "*.agent" } | select-object  FieldValues

     foreach ($file in $files) {

        $fileUrl = $file.FieldValues["FileRef"]
        $fileContent = Get-PnPFile -Url $fileUrl -AsString
        $jsonData = $fileContent | ConvertFrom-Json

        $agentData = [PSCustomObject]@{
                    FileName              = $file.FieldValues["FileLeafRef"]
                    FilePath              = $fileUrl
                    SchemaVersion         = $jsonData.schemaVersion
                    CopilotName           = $jsonData.customCopilotConfig.gptDefinition.name
                    Description           = $jsonData.customCopilotConfig.gptDefinition.description
                    Instructions          = $jsonData.customCopilotConfig.gptDefinition.instructions
                    WelcomeMessage        = $jsonData.customCopilotConfig.conversationStarters.welcomeMessage.text
                    ConversationStarters  = ($jsonData.customCopilotConfig.conversationStarters.conversationStarterList.text -join "; ")
                    Capabilities          = ($jsonData.customCopilotConfig.gptDefinition.capabilities.name -join "; ")
                    ReferencedURLs        = ($jsonData.customCopilotConfig.gptDefinition.capabilities.items_by_url.url -join "; ")
                    SiteIDs               = ($jsonData.customCopilotConfig.gptDefinition.capabilities.items_by_url.site_id -join "; ")
                    WebIDs                = ($jsonData.customCopilotConfig.gptDefinition.capabilities.items_by_url.web_id -join "; ")
                    ListIDs               = ($jsonData.customCopilotConfig.gptDefinition.capabilities.items_by_url.list_id -join "; ")
                    UniqueIDs             = ($jsonData.customCopilotConfig.gptDefinition.capabilities.items_by_url.unique_id -join "; ")
                }

        $allAgentFiles += $agentData

     }
}


$allAgentFiles | Export-Csv -Path $fileLocation -NoTypeInformation
Write-Host $fileLocation

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Valeras Narbutas |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-agent-list" aria-hidden="true" />

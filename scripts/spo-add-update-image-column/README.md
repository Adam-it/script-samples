

# Add/Update Image in SharePoint Image column

## Summary

This sample script shows how to create a new list item with image column and update existing list item to update the image column using PnP PowerShell and CLI for Microsoft 365.

Scenario inspired from this blog post: [Add/Update image columns in SharePoint/Microsoft Lists using PnP PowerShell](https://ganeshsanapblogs.wordpress.com/2022/10/13/add-update-image-columns-in-sharepoint-microsoft-lists-using-pnp-powershell/)

![Outupt Screenshot](assets/output.png)

# [PnP PowerShell](#tab/pnpps)

```powershell

# SharePoint online site URL
$siteUrl = Read-Host -Prompt "Enter your site URL (e.g https://<tenant>.sharepoint.com/sites/contoso)"

# Server Relative URL of image file from same SharePoint site
$serverRelativeUrl = Read-Host -Prompt "Enter Server Relative URL of image file (e.g /sites/contoso/SiteAssets/Lists/dbc6f551-252b-462f-8002-c8f88d0d12d5/PnP-PowerShell-Blue.png)"

# Connect to SharePoint Online site
Connect-PnPOnline -Url $siteUrl -Interactive

# Get UniqueId of file you're referencing (without this part your image won't appear in Power Apps (browser or mobile app) or Microsoft Lists (iOS app))
$imageFileUniqueId = (Get-PnPFile -Url $serverRelativeUrl -AsListItem)["UniqueId"]

# Create new list item with image column
Add-PnPListItem -List "Logo Universe" -Values @{"Title" = "PnP PowerShell"; "Image" = "{'type':'thumbnail','fileName':'PnP-PowerShell-Blue.png','fieldName':'Image','serverUrl':'https://contoso.sharepoint.com','serverRelativeUrl':'$($serverRelativeUrl)', 'id':'$($imageFileUniqueId)'}"}

# Update list item with image column
Set-PnPListItem -List "Logo Universe" -Identity 15 -Values @{"Image" = "{'type':'thumbnail','fileName':'PnP-PowerShell-Blue.png','fieldName':'Image','serverUrl':'https://contoso.sharepoint.com','serverRelativeUrl':'$($serverRelativeUrl)', 'id':'$($imageFileUniqueId)'}"}

# Disconnect SharePoint online connection
Disconnect-PnPOnline
	
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Set-ListImageColumnValue {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory, HelpMessage = "SharePoint site URL hosting the list (e.g. https://contoso.sharepoint.com/sites/site)")]
        [ValidateNotNullOrEmpty()][string] $SiteUrl,

        [Parameter(Mandatory, HelpMessage = "Display name of the SharePoint list containing the image column")]
        [ValidateNotNullOrEmpty()][string] $ListTitle,

        [Parameter(Mandatory, HelpMessage = "Server-relative URL of the source image file within the same site collection")]
        [ValidateNotNullOrEmpty()][string] $ServerRelativeUrl,

        [Parameter(HelpMessage = "Internal name of the image column to update")]
        [ValidateNotNullOrEmpty()][string] $ImageFieldInternalName = 'Image',

        [Parameter(HelpMessage = "Title to use when creating a new list item")]
        [ValidateNotNullOrEmpty()][string] $NewItemTitle,

        [Parameter(HelpMessage = "ID of the existing list item whose image column should be updated")]
        [ValidateRange(1, 2147483647)][int] $ItemIdToUpdate
    )

    begin {
        Write-Verbose "Ensuring CLI session is authenticated."
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to ensure CLI login. CLI output: $loginOutput"
        }

        try {
            $script:SiteUri = [Uri]$SiteUrl
        }
        catch {
            throw "Unable to parse SiteUrl '$SiteUrl'. $($_.Exception.Message)"
        }

        $script:Summary = [ordered]@{
            ItemsCreated = 0
            ItemsUpdated = 0
            Failures     = 0
        }
    }

    process {
        if (-not $PSBoundParameters.ContainsKey('NewItemTitle') -and -not $PSBoundParameters.ContainsKey('ItemIdToUpdate')) {
            Write-Warning "No action requested. Provide -NewItemTitle and/or -ItemIdToUpdate to create or update list items."
            return
        }

        Write-Host "Retrieving image metadata from '$ServerRelativeUrl'."
        $fileOutput = m365 spo file get --webUrl $SiteUrl --url $ServerRelativeUrl --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            $script:Summary.Failures++
            throw "Failed to obtain file metadata. CLI output: $fileOutput"
        }

        try {
            $fileInfo = $fileOutput | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            $script:Summary.Failures++
            throw "Unable to parse file metadata as JSON. $($_.Exception.Message)"
        }

        $serverUrl = "{0}://{1}" -f $script:SiteUri.Scheme, $script:SiteUri.Host
        $imageObject = [ordered]@{
            type             = 'thumbnail'
            fileName         = (Split-Path -Path $ServerRelativeUrl -Leaf)
            fieldName        = $ImageFieldInternalName
            serverUrl        = $serverUrl
            serverRelativeUrl= $ServerRelativeUrl
            id               = $fileInfo.UniqueId
        }

        if ($PSBoundParameters.ContainsKey('NewItemTitle')) {
            if ($PSCmdlet.ShouldProcess("List '$ListTitle'", "Add new item with image")) {
                $fields = [ordered]@{ Title = $NewItemTitle }
                $fields[$ImageFieldInternalName] = $imageObject
                $fieldsJson = $fields | ConvertTo-Json -Compress

                Write-Host "Creating new list item titled '$NewItemTitle'."
                $addOutput = m365 spo listitem add --webUrl $SiteUrl --listTitle $ListTitle --fields $fieldsJson --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    $script:Summary.Failures++
                    Write-Warning "Failed to create list item. CLI output: $addOutput"
                }
                else {
                    $script:Summary.ItemsCreated++
                }
            }
        }

        if ($PSBoundParameters.ContainsKey('ItemIdToUpdate')) {
            if ($PSCmdlet.ShouldProcess("List '$ListTitle' item $ItemIdToUpdate", "Update image column")) {
                $updateFields = [ordered]@{}
                $updateFields[$ImageFieldInternalName] = $imageObject
                $updateFieldsJson = $updateFields | ConvertTo-Json -Compress

                Write-Host "Updating item $ItemIdToUpdate with new image reference."
                $updateOutput = m365 spo listitem set --webUrl $SiteUrl --listTitle $ListTitle --id $ItemIdToUpdate --fields $updateFieldsJson --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    $script:Summary.Failures++
                    Write-Warning "Failed to update list item $ItemIdToUpdate. CLI output: $updateOutput"
                }
                else {
                    $script:Summary.ItemsUpdated++
                }
            }
        }
    }

    end {
        Write-Host "`nOperation summary:" -ForegroundColor Cyan
        Write-Host "  Items created : $($script:Summary.ItemsCreated)"
        Write-Host "  Items updated : $($script:Summary.ItemsUpdated)"
        Write-Host "  Failures      : $($script:Summary.Failures)"
    }
}

# Example usage:
# Set-ListImageColumnValue -SiteUrl "https://contoso.sharepoint.com/sites/contoso" -ListTitle "Logo Universe" -ServerRelativeUrl "/sites/contoso/SiteAssets/Lists/dbc6f551-252b-462f-8002-c8f88d0d12d5/PnP-PowerShell-Blue.png" -NewItemTitle "PnP PowerShell" -ItemIdToUpdate 15

Set-ListImageColumnValue -SiteUrl "https://<tenant>.sharepoint.com/sites/contoso" -ListTitle "Logo Universe" -ServerRelativeUrl "/sites/contoso/SiteAssets/Lists/dbc6f551-252b-462f-8002-c8f88d0d12d5/PnP-PowerShell-Blue.png" -NewItemTitle "PnP PowerShell" -ItemIdToUpdate 15
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/about) |
| [Matt Jimison](https://mattjimison.com) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-add-update-image-column" aria-hidden="true" />

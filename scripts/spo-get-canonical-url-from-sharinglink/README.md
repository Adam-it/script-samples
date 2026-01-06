# Converting SharePoint Sharing Links to Canonical URLs

## Summary

During a recent community discussion, Suhail Sayed presented an interesting challenge that many organizations face during SharePoint tenant migrations. The problem? Converting sharing links to their canonical URLs when documents are migrated between tenants.

---

## The Challenge

When migrating SharePoint documents from one tenant to another, organizations often encounter a specific issue with document references:

### The Problem Scenario:

- **Source tenant**: Documents contain reference links to other documents
- **Migration requirement**: Update these links to reflect the new tenant
- **Complication**: Links were created using the **Share** option, not the **Copy Link** option


### Why This Matters:

When you generate a link using SharePoint's Share function, it creates a unique link with a randomly generated ID that redirects to the correct canonical URL. These unique IDs have no meaning on the new tenant, making simple find-and-replace operations ineffective.

### Example URLs:

**Sharing Link Format:**

```
https://contoso.sharepoint.com/:f:/s/Company311/Et83kyw3weBCqfgt9R73ZVgBDxDRU71gOt1Qkqb99kKubQ
```

**Canonical URL Format:**

```
https://contoso.sharepoint.com/sites/Company311/Shared Documents/Test1/Folder_57/TestDoc_6.docx
```

The script is to extract sharing link information and map them to their corresponding canonical URLs.

### Prerequisites

- The user account that runs the script must have access to the SharePoint Online site.

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding()]
param (
    [Parameter(Mandatory, HelpMessage = "The sharing link URL to resolve (e.g., https://contoso.sharepoint.com/:f:/s/site/...)")]
    [string]$LinkUrl
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = "GetCanonicalUrl_$timestamp.log"
    Start-Transcript -Path $transcriptPath

    Write-Host "[1/5] Parsing sharing link URL..." -ForegroundColor Cyan
    try {
        $uri = [System.Uri]$LinkUrl
        $siteName = $LinkUrl.Split('/')[5]
        $tenantUrl = "$($uri.Scheme)://$($uri.Host)"
        $siteUrl = "$tenantUrl/sites/$($siteName.ToLower())"
        
        Write-Host "Extracted site name: " -NoNewline
        Write-Host $siteName -ForegroundColor Green
        Write-Host "Built site URL: " -NoNewline
        Write-Host $siteUrl -ForegroundColor Green
    } catch {
        throw "Failed to parse sharing link URL: $_"
    }

    Write-Host "`n[2/5] Ensuring CLI for Microsoft 365 login..." -ForegroundColor Cyan
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365. Exit code: $LASTEXITCODE"
    }
    Write-Host "Successfully authenticated." -ForegroundColor Green

    $script:Found = $false
    $script:FileInfo = $null
}

process {
    try {
        Write-Host "`n[3/5] Retrieving document libraries from site..." -ForegroundColor Cyan
        $listsJson = m365 spo list list --webUrl $siteUrl --filter "BaseTemplate eq 101 and Hidden eq false" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve document libraries. CLI: $listsJson"
        }

        $lists = @($listsJson | ConvertFrom-Json)
        Write-Host "Found $($lists.Count) document library(ies) to search." -ForegroundColor Green

        if ($lists.Count -eq 0) {
            Write-Host "No document libraries found in site. Exiting." -ForegroundColor Yellow
            return
        }

        Write-Host "`n[4/5] Searching for file/folder with matching sharing link..." -ForegroundColor Cyan

        foreach ($list in $lists) {
            if ($script:Found) { break }

            Write-Verbose "Searching in library: $($list.Title)"

            try {
                $itemsJson = m365 spo listitem list --webUrl $siteUrl --listTitle $list.Title --fields "FileRef,FileLeafRef,FileSystemObjectType" --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to retrieve items from '$($list.Title)'. CLI: $itemsJson"
                    continue
                }

                $items = @($itemsJson | ConvertFrom-Json)
                Write-Verbose "Processing $($items.Count) item(s) in library '$($list.Title)'"

                foreach ($item in $items) {
                    if ($script:Found) { break }

                    $fileRef = $item.FileRef
                    $fileName = $item.FileLeafRef
                    $isFolder = ($item.FileSystemObjectType -eq 1)

                    try {
                        if ($isFolder) {
                            $sharingLinksJson = m365 spo folder sharinglink list --webUrl $siteUrl --folderUrl $fileRef --output json 2>&1
                        } else {
                            $sharingLinksJson = m365 spo file sharinglink list --webUrl $siteUrl --fileUrl $fileRef --output json 2>&1
                        }

                        if ($LASTEXITCODE -eq 0) {
                            $sharingLinks = @($sharingLinksJson | ConvertFrom-Json)

                            foreach ($link in $sharingLinks) {
                                if ($link.link.webUrl -eq $LinkUrl) {
                                    $script:Found = $true
                                    $script:FileInfo = @{
                                        FileName = $fileName
                                        FileUrl = $fileRef
                                        FullUrl = "$tenantUrl$fileRef"
                                        Library = $list.Title
                                        LinkType = $link.link.type
                                        LinkScope = $link.link.scope
                                        SharingLink = $link.link.webUrl
                                        IsFolder = $isFolder
                                    }

                                    Write-Host "`n✓ Found matching $(if ($isFolder) { 'folder' } else { 'file' })!" -ForegroundColor Green
                                    Write-Host "  Name: $fileName" -ForegroundColor White
                                    Write-Host "  Server-Relative URL: $fileRef" -ForegroundColor White
                                    Write-Host "  Full URL: " -NoNewline
                                    Write-Host "$tenantUrl$fileRef" -ForegroundColor Green
                                    Write-Host "  Library: $($list.Title)" -ForegroundColor White
                                    Write-Host "  Link Type: $($link.link.type)" -ForegroundColor White
                                    Write-Host "  Link Scope: $($link.link.scope)" -ForegroundColor White
                                    break
                                }
                            }
                        }
                    } catch {
                        Write-Verbose "No sharing links found for: $fileRef"
                    }
                }
            } catch {
                Write-Warning "Error searching in library '$($list.Title)': $_"
                continue
            }
        }
    } catch {
        Write-Error "Critical error during execution: $_"
        throw
    }
}

end {
    Write-Host "`n[5/5] Summary..." -ForegroundColor Cyan

    if ($script:Found -and $script:FileInfo) {
        Write-Host "`n========================================" -ForegroundColor Green
        Write-Host "FILE/FOLDER FOUND!" -ForegroundColor Green
        Write-Host "Name: $($script:FileInfo.FileName)" -ForegroundColor White
        Write-Host "Type: $(if ($script:FileInfo.IsFolder) { 'Folder' } else { 'File' })" -ForegroundColor White
        Write-Host "Canonical URL: " -NoNewline
        Write-Host $script:FileInfo.FullUrl -ForegroundColor Cyan
        Write-Host "========================================`n" -ForegroundColor Green
    } else {
        Write-Host "No matching file or folder found for the provided sharing link." -ForegroundColor Yellow
    }

    Write-Host "Transcript log saved to: " -NoNewline
    Write-Host $transcriptPath -ForegroundColor Cyan

    Stop-Transcript
}

# Basic usage
# .\<script>.ps1 -LinkUrl "https://contoso.sharepoint.com/:f:/s/Company311/Et83kyw3weBCqfgt9R73ZVgBDxDRU71gOt1Qkqb99kKubQ"

# With verbose output
# .\<script>.ps1 -LinkUrl "https://contoso.sharepoint.com/:f:/s/Company311/Et83kyw3weBCqfgt9R73ZVgBDxDRU71gOt1Qkqb99kKubQ" -Verbose

# GCC High tenant
# .\<script>.ps1 -LinkUrl "https://contoso.sharepoint.us/:f:/s/Company311/Et83kyw3weBCqfgt9R73ZVgBDxDRU71gOt1Qkqb99kKubQ"

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)

```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$LinkUrl
)

# Extract site name from sharing link
$siteName = $LinkUrl.Split('/')[5]
Write-Host "Extracted site name: $siteName" -ForegroundColor Green
$path = "sites"
# Build site URL from site name
$uri = [System.Uri]$LinkUrl
$tenantUrl = "$($uri.Scheme)://$($uri.Host)"
$siteUrl = "$tenantUrl/$path/$($siteName.ToLower())"
Write-Host "Built site URL: $siteUrl" -ForegroundColor Cyan

connect-PnPOnline -Url $siteUrl 

# Function to find file URL by searching sharing links
function Find-FileUrlByLinkUrl {
    param(
        [string]$SearchLinkUrl
    )
    
    Write-Host "Searching for file URL corresponding to sharing link..." -ForegroundColor Yellow
    
    # Get all lists in the site
    $lists = Get-PnPList | Where-Object { $_.BaseTemplate -eq 101 -and $_.Hidden -eq $false }
    
    # Flag to track if we found a match
    $found = $false
    
    foreach ($list in $lists) {
        if ($found) { break }  # Exit if already found
        
        Write-Host "Searching in library: $($list.Title)" -ForegroundColor Cyan
        
        try {
           # Search folders first
           $Folders= Get-PnPListItem -List $list -Fields "FileRef", "FileLeafRef", "FileSystemObjectType" | Where-Object { $_.FileSystemObjectType -eq "Folder" }
           
           foreach ($folder in $folders) { 
            if ($found) { break }  # Exit if already found
            
            $flinks = Get-PnPFolderSharingLink -Folder $folder['FileRef'] -ErrorAction SilentlyContinue 
                foreach ($flink in $flinks) {
                    # Check if the link URL matches our search URL
                    if ($flink.Link.WebUrl -eq $SearchLinkUrl) {         
                        Write-Host "✓ Found matching folder!" -ForegroundColor Green
                        Write-Host "  Folder Name: $($folder['FileLeafRef'])" -ForegroundColor White
                        Write-Host "  Folder URL: $($folder['FileRef'])" -ForegroundColor White
                        Write-Host "  Full URL: $tenantUrl$($folder['FileRef'])" -ForegroundColor Green
                        Write-Host "  Library: $($list.Title)" -ForegroundColor White
                        Write-Host "  Link Type: $($flink.Link.Type)" -ForegroundColor White
                        Write-Host "  Link Scope: $($flink.Link.Scope)" -ForegroundColor White
                        
                        return @{
                            FileName = $folder['FileLeafRef']
                            FileUrl = $folder['FileRef']
                            FullUrl = "$tenantUrl$($folder['FileRef'])"
                            Library = $list.Title
                            LinkType = $flink.Link.Type
                            LinkScope = $flink.Link.Scope
                            SharingLink = $flink.Link.WebUrl
                        }
                    }
                }
            }

           # Search files only if not found in folders
           if (-not $found) {
               $items  = get-pnplistitem -List $list -Fields "FileRef", "FileLeafRef", "FileSystemObjectType" -PageSize 5000 | Where-Object { $_.FileSystemObjectType -eq "File" }          
                foreach ($item in $items) {
                    if ($found) { break }  # Exit if already found
                    
                    # Get sharing links for each item
                    $sharingLinks = Get-PnPFileSharingLink -Identity $item['FileRef']
                    
                    foreach ($link in $sharingLinks) {
                        # Check if the link URL matches our search URL
                        if ($link.Link.WebUrl -eq $SearchLinkUrl) {
                            
                            Write-Host "✓ Found matching file!" -ForegroundColor Green
                            Write-Host "  File Name: $($item['FileLeafRef'])" -ForegroundColor White
                            Write-Host "  File URL: $($item['FileRef'])" -ForegroundColor White
                            Write-Host "  Full URL: $tenantUrl$($item['FileRef'])" -ForegroundColor Green
                            Write-Host "  Library: $($list.Title)" -ForegroundColor White
                            Write-Host "  Link Type: $($link.Link.Type)" -ForegroundColor White
                            Write-Host "  Link Scope: $($link.Link.Scope)" -ForegroundColor White
                            
                            return @{
                                FileName = $item['FileLeafRef']
                                FileUrl = $item['FileRef']
                                FullUrl = "$tenantUrl$($item['FileRef'])"
                                Library = $list.Title
                                LinkType = $link.Link.Type
                                LinkScope = $link.Link.Scope
                                SharingLink = $link.Link.WebUrl
                            }
                        }
                    }
                }
            }
        }
        catch {
            Write-Host "  Error searching in $($list.Title): $($_.Exception.Message)" -ForegroundColor Red
        }
    }
    
    Write-Host "No matching file found for the provided sharing link." -ForegroundColor Yellow
    return $null
}

# Search for the file corresponding to the sharing link
$fileInfo = Find-FileUrlByLinkUrl -SearchLinkUrl $LinkUrl

if ($fileInfo) {
    Write-Host "`n========================================" -ForegroundColor Green
    Write-Host "FILE FOUND!" -ForegroundColor Green
    Write-Host "File: $($fileInfo.FileName)" -ForegroundColor White
    Write-Host "URL: $($fileInfo.FullUrl)" -ForegroundColor White
    Write-Host "========================================`n" -ForegroundColor Green
}
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Source Credit

Sample first appeared on [Converting SharePoint Sharing Links to Canonical URLs with PowerShell](https://reshmeeauckloo.com/posts/powershell-sharepoint-getfileurl-basedon-sharingurl/)

## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Reshmee Auckloo](https://github.com/reshmee011) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-get-canonical-url-from-sharinglink" aria-hidden="true" />



# Export and import library folder structure

## Summary

Sometimes you just need to copy a folder structure from one library to another. This script will export the folder structure from one library and import it to another library using a JSON file to store the folder structure. This can be used as is or be an individual function in a site provisioning script.

The CLI for Microsoft 365 implementation uses modern PowerShell practices with typed parameters, parameter sets for export/import modes, WhatIf support, and comprehensive error handling. It leverages CLI v11.4.0+ commands including recursive folder listing, native folder color support, and automatic parent folder creation. The script exports folder hierarchies with color preservation to JSON and imports them with atomic folder creation operations.


![Example Screenshot](assets/example.png)


# [PnP PowerShell](#tab/pnpps)

```powershell
# export part 

function Get-Folderstructure 
{
    param(
        [Parameter(Mandatory=$true)]
        [string]$folderUrl,
        [Parameter(Mandatory=$true)]
        [string]$folderName,
        [Parameter(Mandatory=$true)]
        [string]$ListName,
        [Parameter(Mandatory=$true)]
        [int]$level
        
    )
   # $thisFolder = Get-PnPFolder -Url $folderUrl -Connection $conn
   if($folderName -eq "Forms")
   {
        continue 
   }
    $result = $null
    $folderColl=Get-PnPFolderItem -FolderSiteRelativeUrl $folderUrl -ItemType Folder -Connection $conn  
    $leaves = @()
    $elements = @()
    foreach($folder in $folderColl)
    {
        
        $subFolderURL= $folderUrl+"/"+$folder.Name
        $testForSubFolders=Get-PnPFolderItem -FolderSiteRelativeUrl $subFolderURL -ItemType Folder -Connection $conn  
        if($testForSubFolders.Count -gt 0)
        {
            write-host "subfolder found for folder $($folder.Name) at level $level" -ForegroundColor Yellow
            $level = $level+1
            $result = Get-Folderstructure -folderUrl $subFolderURL -ListName $ListName -folderName $folder.Name -level $level            
            $result = $result | ConvertFrom-Json -Depth 100
            $elements += $result

        }
        else 
        {
            #Write-Host "leaf node $($folder.Name)" -ForegroundColor Yellow
            $FolderItem = Get-PnPListItem -List $ListName -UniqueId $folder.UniqueId -ErrorAction Stop -Connection $conn
            $foldercolor = $FolderItem.FieldValues["_ColorHex"]
            if($foldercolor)
            {
                Write-Host "color found for folder $($folder.Name) - $foldercolor" -ForegroundColor Green
            }
            $leave = [PSCustomObject]@{Name = $folder.Name; Color = $foldercolor}    
            $elements += $leave
        }
        
        
    }
    if($elements.Count -gt 0)
    {

        $folder = Get-PnPFolder -Url $folderUrl -Connection $conn
        $UniqueId = Get-PnPProperty -ClientObject $folder -Property UniqueId -Connection $conn
        
        $FolderItem = Get-PnPListItem -List $ListName -UniqueId $UniqueId -ErrorAction Stop -Connection $conn
        $foldercolor = $FolderItem.FieldValues["_ColorHex"]
        if($foldercolor)
        {
            Write-Host "color found for folder $($folder.Name) - $foldercolor" -ForegroundColor Green
        }
        $element = [PSCustomObject]@{
            Name = $folderName
            Color = $foldercolor
            Folders = $elements
        }
    }
    
    $element = $element | ConvertTo-Json -Depth 100
    return $element
}
$url = "https://contoso.sharepoint.com/sites/thesite"
$conn = Connect-PnPOnline -Url $url -Interactive -ReturnConnection

$finalJson = @()
$finalJson = Get-Folderstructure -folderUrl "/Shared Documents"
$finalJson | ConvertTo-Json -Depth 100
#write result to file
$finalJson | Out-File -FilePath "C:\temp\folderstructureRecursive.json" -Force

################################
# import part

function SetFolder 
{
    #add mandatory parameter folder
    param(
        [Parameter(Mandatory=$true)]
        [object]$folder,
        [Parameter(Mandatory=$true)]
        [string]$folderurl,
        [Parameter(Mandatory=$true)]
        [string]$documentLibrary
    )
    $createdFolder = Add-PnPFolder -Name $folder.Name -Folder $folderurl -ErrorAction Stop -Connection $conn    
    # Get the created folder item
    $newFolderItem = Get-PnPListItem -List $documentLibrary -UniqueId $createdFolder.UniqueId -ErrorAction Stop -Connection $conn
    $folderColor = $folder.Color
    # Change the value of the _ColorHex column of the created folder to change the color
    Set-PnPListItem -List $documentLibrary -Identity $newFolderItem.Id -Values @{"_ColorHex" = $folderColor } -ErrorAction Stop  -Connection $conn  

    foreach($folder in $folder.Folders)
    {
        SetFolder -folder $folder -folderurl $createdFolder.ServerRelativeUrl -documentLibrary $documentLibrary
    }
}
    


function Set-FolderstructurefromJson 
{
    param(
        [Parameter(Mandatory=$true)]
        [object]$json,
        [Parameter(Mandatory=$true)]
        [string]$folderName,
        [Parameter(Mandatory=$true)]
        [string]$documentLibrary
    )
    $folders = $json.Folders

    foreach($folder in $folders)
    {
        $res = SetFolder -folder $folder -folderurl $folderName -documentLibrary $documentLibrary 
    }

}

$url = "https://contoso.sharepoint.com/sites/targetsite/"
$conn = connect-pnponline -URL $url -Interactive -ReturnConnection

#the json file can be stored in a document library or locally
$jsonUrl = "/sites/SampleTeamSite/Shared%20Documents/folderstructureRecursive.json"
$jsonSiteUrl = "https://contoso.sharepoint.com/sites/SampleTeamSite"
$jsonsiteconn = Connect-PnPOnline -Url $jsonSiteUrl  -Interactive -ReturnConnection

$file = Get-PnPFile -Url $jsonUrl -AsString -Connection $jsonsiteconn
$json = ConvertFrom-Json $file 

#if you want start the folder structure in the root of the library
Set-FolderstructurefromJson -json $json -folderName "Shared Documents/"

#if you want to start the folder structure in a subfolder of the library
Set-FolderstructurefromJson -json $json -folderName "Shared Documents/General"


# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory = $true, ParameterSetName = 'Export', HelpMessage = "Source site URL for export")]
    [Parameter(Mandatory = $true, ParameterSetName = 'Import', HelpMessage = "Target site URL for import")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)')]
    [string]$SiteUrl,
    
    [Parameter(Mandatory = $true, ParameterSetName = 'Export', HelpMessage = "Document library name to export from")]
    [Parameter(Mandatory = $true, ParameterSetName = 'Import', HelpMessage = "Document library name to import to")]
    [ValidateNotNullOrEmpty()]
    [string]$LibraryName,
    
    [Parameter(Mandatory = $true, ParameterSetName = 'Export', HelpMessage = "Path to save exported JSON file")]
    [ValidateScript({ Test-Path (Split-Path $_) -PathType Container })]
    [string]$ExportPath,
    
    [Parameter(Mandatory = $true, ParameterSetName = 'Import', HelpMessage = "Path to JSON file with folder structure")]
    [ValidateScript({ Test-Path $_ -PathType Leaf })]
    [string]$ImportPath,
    
    [Parameter(Mandatory = $false, ParameterSetName = 'Import', HelpMessage = "Target folder path within library (default: library root)")]
    [string]$TargetFolder = ""
)

begin {
    $script:Summary = @{
        Mode = if ($PSCmdlet.ParameterSetName -eq 'Export') { 'Export' } else { 'Import' }
        FoldersProcessed = 0
        FoldersCreated = 0
        Failures = 0
    }
    
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $transcriptPath = Join-Path ([System.IO.Path]::GetTempPath()) "FolderStructure-$($script:Summary.Mode)-$timestamp.log"
    Start-Transcript -Path $transcriptPath
    
    Write-Host "Folder Structure $($script:Summary.Mode) Tool" -ForegroundColor Cyan
    Write-Host "Site URL: $SiteUrl" -ForegroundColor White
    Write-Host "Library: $LibraryName" -ForegroundColor White
    
    Write-Verbose "Ensuring login to Microsoft 365"
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        Stop-Transcript
        throw "Failed to login to Microsoft 365. Please run 'm365 login' first."
    }
    
    Write-Verbose "Verifying library exists"
    $library = m365 spo list get --webUrl $SiteUrl --title $LibraryName --output json 2>&1 | ConvertFrom-Json
    if ($LASTEXITCODE -ne 0 -or -not $library) {
        Stop-Transcript
        throw "Library '$LibraryName' not found at site '$SiteUrl'"
    }
    
    $script:LibraryServerRelativeUrl = $library.RootFolder.ServerRelativeUrl
    Write-Verbose "Library URL: $($script:LibraryServerRelativeUrl)"
}

process {
    if ($PSCmdlet.ParameterSetName -eq 'Export') {
        Write-Host "`nExporting folder structure..." -ForegroundColor Yellow
        
        try {
            Write-Verbose "Fetching folders recursively from $($script:LibraryServerRelativeUrl)"
            $folders = m365 spo folder list --webUrl $SiteUrl --parentFolderUrl $script:LibraryServerRelativeUrl --recursive --output json 2>&1 | ConvertFrom-Json
            
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to list folders: $folders"
            }
            
            $folders = @($folders)
            Write-Host "Found $($folders.Count) folder$(if ($folders.Count -ne 1) { 's' })" -ForegroundColor Green
            
            function Build-FolderTree {
                param(
                    [array]$AllFolders,
                    [string]$ParentPath
                )
                
                $children = $AllFolders | Where-Object { 
                    $parentUrl = Split-Path $_.ServerRelativeUrl -Parent
                    $parentUrl -eq $ParentPath
                }
                
                $result = @()
                foreach ($folder in $children) {
                    $script:Summary.FoldersProcessed++
                    Write-Progress -Activity "Processing folders" -Status "$($script:Summary.FoldersProcessed)/$($AllFolders.Count): $($folder.Name)" -PercentComplete (($script:Summary.FoldersProcessed / $AllFolders.Count) * 100)
                    
                    Write-Verbose "Processing folder: $($folder.Name)"
                    
                    $folderColor = $null
                    try {
                        $folderDetails = m365 spo folder get --webUrl $SiteUrl --url $folder.ServerRelativeUrl --output json 2>&1 | ConvertFrom-Json
                        if ($LASTEXITCODE -eq 0 -and $folderDetails.ListItemAllFields) {
                            $colorHex = $folderDetails.ListItemAllFields.'_ColorHex'
                            $colorTag = $folderDetails.ListItemAllFields.'OData__ColorTag'
                            $folderColor = if ($colorTag) { $colorTag } elseif ($colorHex) { $colorHex } else { $null }
                            if ($folderColor) {
                                Write-Verbose "Found color for '$($folder.Name)': $folderColor"
                            }
                        }
                    }
                    catch {
                        Write-Verbose "Could not retrieve color for folder '$($folder.Name)': $_"
                    }
                    
                    $subFolders = Build-FolderTree -AllFolders $AllFolders -ParentPath $folder.ServerRelativeUrl
                    
                    $folderObj = [PSCustomObject]@{
                        Name = $folder.Name
                        Color = $folderColor
                    }
                    
                    if ($subFolders.Count -gt 0) {
                        $folderObj | Add-Member -MemberType NoteProperty -Name Folders -Value $subFolders
                    }
                    
                    $result += $folderObj
                }
                
                return $result
            }
            
            Write-Host "Building folder hierarchy..." -ForegroundColor Yellow
            $folderTree = Build-FolderTree -AllFolders $folders -ParentPath $script:LibraryServerRelativeUrl
            
            Write-Progress -Activity "Processing folders" -Completed
            
            if ($PSCmdlet.ShouldProcess($ExportPath, "Export folder structure to JSON file")) {
                $jsonContent = [PSCustomObject]@{
                    ExportDate = (Get-Date).ToString('yyyy-MM-dd HH:mm:ss')
                    SiteUrl = $SiteUrl
                    LibraryName = $LibraryName
                    Folders = $folderTree
                } | ConvertTo-Json -Depth 100
                
                $jsonContent | Out-File -FilePath $ExportPath -Encoding UTF8 -Force
                Write-Host "Exported folder structure to: $ExportPath" -ForegroundColor Green
            }
        }
        catch {
            Write-Error "Export failed: $_"
            $script:Summary.Failures++
        }
    }
    else {
        Write-Host "`nImporting folder structure..." -ForegroundColor Yellow
        
        try {
            Write-Verbose "Reading JSON file: $ImportPath"
            $jsonContent = Get-Content -Path $ImportPath -Raw -Encoding UTF8 | ConvertFrom-Json
            
            $foldersToImport = $jsonContent.Folders
            if (-not $foldersToImport) {
                throw "Invalid JSON structure. Expected 'Folders' property."
            }
            
            $baseUrl = if ($TargetFolder) {
                "$($script:LibraryServerRelativeUrl)/$($TargetFolder.TrimStart('/'))"
            } else {
                $script:LibraryServerRelativeUrl
            }
            
            Write-Host "Target location: $baseUrl" -ForegroundColor White
            
            function Import-Folders {
                param(
                    [array]$Folders,
                    [string]$ParentUrl
                )
                
                foreach ($folder in $Folders) {
                    $script:Summary.FoldersProcessed++
                    Write-Progress -Activity "Creating folders" -Status "$($script:Summary.FoldersProcessed): $($folder.Name)" -PercentComplete (($script:Summary.FoldersProcessed / ($script:Summary.FoldersProcessed + 1)) * 100)
                    
                    try {
                        if ($PSCmdlet.ShouldProcess("$ParentUrl/$($folder.Name)", "Create folder with color: $($folder.Color)")) {
                            Write-Verbose "Creating folder: $($folder.Name) in $ParentUrl"
                            
                            $args = @(
                                'spo', 'folder', 'add',
                                '--webUrl', $SiteUrl,
                                '--parentFolderUrl', $ParentUrl,
                                '--name', $folder.Name,
                                '--output', 'json'
                            )
                            
                            if ($folder.Color) {
                                $args += '--color'
                                $args += $folder.Color
                            }
                            
                            $newFolder = m365 @args 2>&1 | ConvertFrom-Json
                            
                            if ($LASTEXITCODE -ne 0) {
                                throw "CLI command failed"
                            }
                            
                            $script:Summary.FoldersCreated++
                            Write-Host "Created folder: $($folder.Name)" -ForegroundColor Green
                            
                            if ($folder.Folders -and $folder.Folders.Count -gt 0) {
                                Import-Folders -Folders $folder.Folders -ParentUrl $newFolder.ServerRelativeUrl
                            }
                        }
                    }
                    catch {
                        Write-Warning "Failed to create folder '$($folder.Name)': $_"
                        $script:Summary.Failures++
                        continue
                    }
                }
            }
            
            Import-Folders -Folders $foldersToImport -ParentUrl $baseUrl
            Write-Progress -Activity "Creating folders" -Completed
        }
        catch {
            Write-Error "Import failed: $_"
            $script:Summary.Failures++
        }
    }
}

end {
    Write-Host "`n=== Folder Structure $($script:Summary.Mode) Summary ===" -ForegroundColor Cyan
    Write-Host "Mode: $($script:Summary.Mode)" -ForegroundColor White
    Write-Host "Folders Processed: $($script:Summary.FoldersProcessed)" -ForegroundColor White
    
    if ($script:Summary.Mode -eq 'Import') {
        $createdColor = if ($script:Summary.FoldersCreated -gt 0) { "Green" } else { "Yellow" }
        Write-Host "Folders Created: $($script:Summary.FoldersCreated)" -ForegroundColor $createdColor
    }
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures: $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures: 0" -ForegroundColor Green
    }
    
    Write-Host "==========================================`n" -ForegroundColor Cyan
    Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Gray
    
    Stop-Transcript
}

# Example 1: Export folder structure to JSON file
# .\Export-Import-FolderStructure.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/source" -LibraryName "Documents" -ExportPath "C:\temp\folders.json"

# Example 2: Import folder structure to library root
# .\Export-Import-FolderStructure.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/target" -LibraryName "Documents" -ImportPath "C:\temp\folders.json"

# Example 3: Import to subfolder with WhatIf
# .\Export-Import-FolderStructure.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/target" -LibraryName "Documents" -ImportPath "C:\temp\folders.json" -TargetFolder "Projects/2024" -WhatIf

# Example 4: Export with verbose output
# .\Export-Import-FolderStructure.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/source" -LibraryName "Documents" -ExportPath "C:\temp\folders.json" -Verbose

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| Kasper Larsen |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-import-folderstructure" aria-hidden="true" />

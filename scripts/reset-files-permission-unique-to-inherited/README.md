

# Reset files permissions unique to Inherited

## Summary
Reset bulk file permissions  from unique to parent folder inheritance.

# [PnP PowerShell](#tab/pnpps)
```powershell

# Make sure necessary modules are installed
# PnP PowerShell to get access to M365 tenent

Install-Module PnP.PowerShell
$siteURL = "https://tenent.sharepoint.com/sites/Dataverse"
Connect-PnPOnline -Url $siteURL -Credentials (Get-Credential)
$listName = "Document Library"
#Get the Context
$Context = Get-PnPContext

try {
    ## Get all folders from given list
    $folders = Get-PnPFolder -List $listName
}
catch {
    ## Do this if a terminating exception happens
    Write-Host "Error: $($_.Exception.Message)" -ForegroundColor Red
    try {
        Write-Host "Trying to use Get-PnPListItem" -ForegroundColor Yellow
        #Treat the folder as item, and the item attribute is Folder (FileSystemObjectType -eq "Folder")  
    $folders = Get-PnPListItem -List $listName -PageSize 500 -Fields FileLeafRef | Where {$_.FileSystemObjectType -eq "Folder"}
    }
    catch {
        Write-Host "Error: $($_.Exception.Message)" -ForegroundColor Red
    }
}

Write-Output "Total Folder found $($folders.Count)"
## Traverse all files from all folders.
foreach($folder in $folders){
    Write-Host "get all files from folder '$($folder.Name)'" -ForegroundColor DarkGreen
    $files = Get-PnPListItem -List $listName -FolderServerRelativeUrl $folder.ServerRelativeUrl -PageSize 500 
    Write-Host "Total Files found $($Files.Count) in folder $($folder.Name)" -ForegroundColor DarkGreen
    foreach ($file in $files){
        ## Check object type is file or folder.If file than do process else do nothing.
        if($file.FileSystemObjectType.ToString() -eq "File"){
            #Check File is unique permission or inherited permission.
            # If File has Unique Permission than below line return True else False
            $hasUniqueRole = Get-PnPProperty -ClientObject $file -Property HasUniqueRoleAssignments
            if($hasUniqueRole -eq $true){
                ## If File has Unique Permission than reset it to inherited permission from parent folder.
                Write-Host "Reset Permisison starting for file with id $($file.Id)" -ForegroundColor DarkGreen
                $file.ResetRoleInheritance()
                $file.update()
                $Context.ExecuteQuery()
            }
        }
    }
}
## Disconnect PnP Connection.
Disconnect-PnPOnline
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]


# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
function Restore-ListItemInheritance {
    [CmdletBinding(SupportsShouldProcess = $true)]
    param (
        [Parameter(Mandatory = $true, HelpMessage = "URL of the SharePoint site" )]
        [ValidateNotNullOrEmpty()]
        [string] $SiteUrl,

        [Parameter(Mandatory = $true, HelpMessage = "Title of the document library" )]
        [ValidateNotNullOrEmpty()]
        [string] $LibraryTitle,

        [Parameter(HelpMessage = "Folder server-relative URL. When omitted the script scans from the list root." )]
        [string] $FolderServerRelativeUrl,

        [Parameter(HelpMessage = "Recursively process sub folders" )]
        [switch] $Recursive,

        [Parameter(HelpMessage = "Optional path to export processed items and status" )]
        [string] $OutputPath,

        [Parameter(HelpMessage = "Emit the processed items to the pipeline" )]
        [switch] $PassThru
    )

    begin {
        Write-Verbose "Ensuring CLI authentication"
        m365 login --ensure | Out-Null

        $results = New-Object System.Collections.Generic.List[psobject]
        $summary = [ordered]@{
            ItemsScanned    = 0
            ItemsReset      = 0
            ItemsInherited  = 0
            Errors          = 0
        }

        if ($OutputPath) {
            $directory = Split-Path -Parent $OutputPath
            if (-not $directory) {
                $directory = '.'
            }
            if (-not (Test-Path -Path $directory)) {
                Write-Verbose "Creating directory '$directory'"
                New-Item -ItemType Directory -Path $directory -Force | Out-Null
            }
        }

        $list = $null
        $listArgs = @(
            'spo', 'list', 'get',
            '--webUrl', $SiteUrl,
            '--title', $LibraryTitle,
            '--output', 'json',
            '--query', '{Id:Id,Title:Title,RootFolderServerRelativeUrl:RootFolder.ServerRelativeUrl}'
        )

        Write-Verbose "Retrieving list metadata"
        $listJson = m365 @listArgs 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve list '$LibraryTitle'. CLI output: $listJson"
        }

        try {
            $list = $listJson | ConvertFrom-Json
        }
        catch {
            throw "Failed to parse list metadata. Error: $($_.Exception.Message)"
        }

        if (-not $FolderServerRelativeUrl) {
            $FolderServerRelativeUrl = $list.RootFolderServerRelativeUrl
        }

        function Invoke-ItemRepair {
            param(
                [string] $Category,
                [string] $ItemServerRelativeUrl,
                [string] $FileId
            )

            $summary.ItemsScanned++

            $itemArgs = @(
                'spo', 'listitem', 'get',
                '--webUrl', $SiteUrl,
                '--listId', $list.Id,
                '--id', $FileId,
                '--properties', 'HasUniqueRoleAssignments',
                '--output', 'json'
            )

            $itemJson = m365 @itemArgs 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve list item '$ItemServerRelativeUrl'. CLI output: $itemJson"
                $summary.Errors++
                return
            }

            try {
                $item = $itemJson | ConvertFrom-Json
            }
            catch {
                Write-Warning "Failed to parse list item '$ItemServerRelativeUrl'. Error: $($_.Exception.Message)"
                $summary.Errors++
                return
            }

            $resultEntry = [pscustomobject]@{
                ServerRelativeUrl = $ItemServerRelativeUrl
                ItemId            = $item.Id
                HadUniquePermission = $item.HasUniqueRoleAssignments
                ResetPerformed    = $false
            }

            if ($item.HasUniqueRoleAssignments) {
                if ($PSCmdlet.ShouldProcess($ItemServerRelativeUrl, 'Restore permission inheritance')) {
                    $resetArgs = @(
                        'spo', 'listitem', 'roleinheritance', 'reset',
                        '--webUrl', $SiteUrl,
                        '--listId', $list.Id,
                        '--listItemId', $item.Id
                    )

                    $resetOutput = m365 @resetArgs 2>&1
                    if ($LASTEXITCODE -ne 0) {
                        Write-Warning "Failed to reset inheritance for '$ItemServerRelativeUrl'. CLI output: $resetOutput"
                        $summary.Errors++
                    }
                    else {
                        $summary.ItemsReset++
                        $resultEntry.ResetPerformed = $true
                    }
                }
            }
            else {
                $summary.ItemsInherited++
            }

            $results.Add($resultEntry) | Out-Null
        }

    }

    process {
        $fileArgs = @(
            'spo', 'file', 'list',
            '--webUrl', $SiteUrl,
            '--folderUrl', $FolderServerRelativeUrl,
            '--output', 'json',
            '--query', '[].{ServerRelativeUrl:ServerRelativeUrl,UniqueId:UniqueId}'
        )

        if ($Recursive) {
            $fileArgs += '--recursive'
        }

        $filesJson = m365 @fileArgs 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to list files under '$FolderServerRelativeUrl'. CLI output: $filesJson"
            $summary.Errors++
            return
        }

        if ([string]::IsNullOrWhiteSpace($filesJson)) {
            Write-Verbose "No files returned for '$FolderServerRelativeUrl'"
            return
        }

        try {
            $files = @($filesJson | ConvertFrom-Json)
        }
        catch {
            Write-Warning "Failed to parse file listing for '$FolderServerRelativeUrl'. Error: $($_.Exception.Message)"
            $summary.Errors++
            return
        }

        foreach ($file in $files) {
            Invoke-ItemRepair -Category 'File' -ItemServerRelativeUrl $file.ServerRelativeUrl -FileId $file.UniqueId
        }
    }

    end {
        if ($OutputPath) {
            Write-Verbose "Exporting results to '$OutputPath'"
            $results | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8 -Force
        }

        Write-Host "Reset inheritance summary:" -ForegroundColor Cyan
        Write-Host ("- Items scanned: {0}" -f $summary.ItemsScanned)
        Write-Host ("- Items reset : {0}" -f $summary.ItemsReset)
        Write-Host ("- Already inherited: {0}" -f $summary.ItemsInherited)
        Write-Host ("- Errors: {0}" -f $summary.Errors)

        if ($PassThru) {
            $results
        }
    }
}

# Example
Restore-ListItemInheritance -SiteUrl "https://contoso.sharepoint.com/sites/Docs" -LibraryTitle "Documents" -Recursive -OutputPath "./RestoredPermissions.csv" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| [Dipen Shah](https://github.com/dips365) |
| [Nanddeep Nachan](https://github.com/nanddeepn) |
| [Valeras Narbutas](https://github.com/ValerasNarbutas) |
| [Rob Ellis](https://github.com/ee61re) |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/reset-files-permission-unique-to-inherited" aria-hidden="true" />

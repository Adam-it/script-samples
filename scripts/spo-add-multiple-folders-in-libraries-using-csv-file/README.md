

# Add multiple folders in libraries using a CSV file

## Summary

This script sample will allow you to create the folders (not nested) into the SharePoint libraries provided in the CSV file.

### Create folders on single SharePoint site

Below is an example of the format needed for your `.csv` file:

| libName | folderName |
| --------| ---------- |
| Customers | Contracts |
| Support | Roadmaps |
| Support | Analysis |
 
> [!important]
> Make sure your target libraries contained in the file do exist in SharePoint Online site.

# [PnP PowerShell](#tab/pnpps)

```powershell
# Config Variables
$SiteURL = "https://contoso.sharepoint.com/sites/Ops"
$CSVFilePath = "C:\Temp\Folders.csv"
 
Try {
    # Connect to PnP Online
    Connect-PnPOnline -Url $SiteURL -Interactive
    $Web = Get-PnPWeb
 
    # Get the CSV file
    $CSVFile = Import-Csv $CSVFilePath
  
    # Read CSV file and create folders
    ForEach($Row in $CSVFile)
    {
        # Get the Document Library and its site relative URL
        $Library = Get-PnPList -Identity $Row.libName -Includes RootFolder

        If($Web.ServerRelativeUrl -eq "/")
        {
            $LibrarySiteRelativeURL = $Library.RootFolder.ServerRelativeUrl
        }
        else
        {
            $LibrarySiteRelativeURL = $Library.RootFolder.ServerRelativeUrl.Replace($Web.ServerRelativeUrl,'')
        }

        # Replace Invalid Characters from Folder Name, If any
        $FolderName = $Row.folderName
        $FolderName = [RegEx]::Replace($FolderName, "[{0}]" -f ([RegEx]::Escape([String]'\"*:<>?/\|')), '_')
 
        # Frame the Folder Name
        $FolderURL = $LibrarySiteRelativeURL+"/"+$FolderName
 
        # Create Folder if it doesn't exist
        Resolve-PnPFolder -SiteRelativePath $FolderURL | Out-Null
        Write-host "Ensured Folder:"$FolderName -f Green
    }
}
catch {
    write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
}
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Add-FoldersFromCsv {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory, HelpMessage = "Path to the CSV file with libName and folderName columns")]
        [ValidateScript({ Test-Path $_ -PathType Leaf })]
        [string]
        $CsvPath,

        [Parameter(Mandatory, HelpMessage = "SharePoint site URL (e.g. https://contoso.sharepoint.com/sites/Ops)")]
        [ValidateNotNullOrEmpty()]
        [string]
        $SiteUrl,

        [Parameter(HelpMessage = "Treat the CSV as UTF8 without BOM")]
        [switch]
        $AsUtf8
    )

    begin {
        Write-Host "Ensuring Microsoft 365 CLI session..." -ForegroundColor Cyan
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to ensure CLI login. CLI output: $loginOutput"
        }

        $script:Summary = [ordered]@{
            RowsProcessed  = 0
            FoldersCreated = 0
            Failures       = 0
        }

        $encoding = if ($AsUtf8.IsPresent) { [System.Text.Encoding]::UTF8 } else { [System.Text.Encoding]::Default }
        $script:Reader = [System.IO.StreamReader]::new($CsvPath, $encoding)
        $script:Headers = $null
    }

    process {
        while (-not $script:Reader.EndOfStream) {
            $line = $script:Reader.ReadLine()
            if (-not $line) { continue }

            if (-not $script:Headers) {
                $script:Headers = $line.Split(',')
                continue
            }

            $values = $line.Split(',')
            $row = [ordered]@{}
            for ($index = 0; $index -lt $script:Headers.Length; $index++) {
                $row[$script:Headers[$index]] = if ($index -lt $values.Length) { $values[$index].Trim() } else { $null }
            }

            $libName = $row["libName"]
            $folderName = $row["folderName"]

            if (-not $libName -or -not $folderName) {
                Write-Warning "Skipping row with missing libName or folderName."
                continue
            }

            $safeFolderName = [RegEx]::Replace($folderName, "[{0}]" -f ([RegEx]::Escape([String]'\"*:<>?/\|')), '_')
            $script:Summary.RowsProcessed++

            $listOutput = m365 spo list get --webUrl $SiteUrl --title $libName --properties "Title,RootFolder/ServerRelativeUrl" --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Failures++
                Write-Warning "Failed to resolve library '$libName'. CLI output: $listOutput"
                continue
            }

            try {
                $library = $listOutput | ConvertFrom-Json -ErrorAction Stop
            }
            catch {
                $script:Summary.Failures++
                Write-Warning "Failed to parse library response for '$libName'. $($_.Exception.Message)"
                continue
            }

            $parentUrl = $library.RootFolder.ServerRelativeUrl
            $targetPath = "$parentUrl/$safeFolderName"

            if (-not $PSCmdlet.ShouldProcess($targetPath, "Create folder")) {
                continue
            }

            $folderOutput = m365 spo folder add --webUrl $SiteUrl --parentFolderUrl $parentUrl --name $safeFolderName --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                $script:Summary.Failures++
                Write-Warning "Failed to create folder '$safeFolderName' in '$libName'. CLI output: $folderOutput"
            }
            else {
                $script:Summary.FoldersCreated++
                Write-Host "Created folder $safeFolderName in $libName" -ForegroundColor Green
            }
        }
    }

    end {
        $script:Reader.Close()

        Write-Host "----- Summary -----" -ForegroundColor Cyan
        Write-Host "Rows processed   : $($script:Summary.RowsProcessed)"
        Write-Host "Folders created  : $($script:Summary.FoldersCreated)"
        Write-Host "Failures         : $($script:Summary.Failures)"

        if ($script:Summary.Failures -gt 0) {
            Write-Warning "Some operations failed. Review warnings above."
        }
    }
}

Add-FoldersFromCsv -CsvPath "D:\\dtemp\\Folders.csv" -SiteUrl "https://contoso.sharepoint.com/sites/Ops" -AsUtf8
# The CSV should contain libName and folderName columns.
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Ganesh Sanap](https://ganeshsanapblogs.wordpress.com/about) |
| [Jiten Parmar](https://github.com/Jitenparmar) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]

<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-add-multiple-folders-in-libraries-using-csv-file" aria-hidden="true" />

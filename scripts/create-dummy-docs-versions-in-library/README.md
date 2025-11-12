

# Create bulk dummy documents, including minor/major versions, in SharePoint Document library

## Summary

Based on a script by Siddharth Vaghasia : scripts/create-dummy-docs-in-library
There are times when we have to replicate scenario to bulk upload dummy documents in large numbers for replicating 5000 items limit or testing performance of dec/test/uat enviorments. This script would help us create 'n' number of dummy documents specified as maxCount in script. Script will also provide option to create dummy folder first for each file and then upload file inside that folder. Script will use the specified file and add counter inside file name to provide uniqueness of file.
The reason for adding a number of versions for each file could be to use it as a testbed for other scripts. In my case I was testing the effect on SharePoint storage costs when stripping versions when a site collection is archived.

new functionality:
For the File option you can specify a number of minor versions you wish to create : $minorVersionCount
$minorVersionCountBeforeMajor specifies how often a major version should be created.

Sample: 
$minorVersionCount = 10
$minorVersionCountBeforeMajor = 3

the version history will be like:
0.1
0.2
1.0
1.1
1.2
2.0
2.1
2.2
3.0
3.1


Note about two available options
- Upload the dummy files directly on the SP library, you can provide this path in "$Folder"
- Create a dummy folder first and upload the file inside that folder, you can provide the root path in "$SiteRelativeURL"

## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as below,
- Change the variables to target to your environment, site, document library, document path, max count
- Run the script.
 
## Screenshot of Output 

 Below is the output after I have ran the script twice with maxCount set to 5, 

- Input as Folder (it has created five folder with auto incrementing folder name to get uniqueness and then added file inside each folder)
- Input as File  (it has created five files and auto incremented file name to get uniqueness)

![Example Screenshot](assets/preview.png)

# [PnP PowerShell](#tab/pnpps)
```powershell

function ensureLibraryIsUsingMinorVersions
{
    Set-PnPList -Identity $Folder -EnableMinorVersions $true
}
#Global Variable Declaration

$pnpPowerShellModule = Get-Module PnP.PowerShell
if ($null -eq $pnpPowerShellModule) {
    Install-Module PnP.PowerShell
}

#Global Variables 
$SiteURL = "https://yourdomain.sharepoint.com/sites/mytestsite" 

#Serverrelative url of the Library, this will be used for Folder scenario
$SiteRelativeURL= "/sites/mytestsite/Shared Documents"

#Local file path where a single dummy document is available

$File= "D:\SP\repos\myscriptsamples\Dummy.docx"


#This can be used for file scenario and provide the folder path where we want to create files, it can be subfolder also
$Folder="Shared Documents"

#Read Information to get which operation need to perform
$MethodCall=Read-Host "Which Function Do you Need to Invoke ? Folder or File" 
$MethodCall=$MethodCall.ToUpper() 

#This will be max count of dummy folder or files which we have to create
$maxCount = 15
#this will define how many minor versions the script should create 
$minorVersionCount = 6

#this will define how many minor versions the script should create before a major version is added
$minorVersionCountBeforeMajor = 3


if($maxCount -lt $minorVersionCount)
{
    throw "MaxCount must be higher than minorVersionCount"
}

#For Sample Document Creation the file needs to be part of some location.
$FilePath= Get-ChildItem $File  
$FileName = $FilePath.BaseName #Inorder to get the filename for the manipulation we used this function(BaseName)

#For Logging the Operations
$LogTime = Get-Date -Format "MM-dd-yyyy_hh-mm-ss"
$LogFile = 'D:\SP\repos\myscriptsamples\'+"FileFolderCreation_"+$LogTime+".txt"


 if($MethodCall -eq "FOLDER" -or $MethodCall -eq "FILE")
 {

 Try 
{
    #Connect to PnP Online
    Connect-PnPOnline -Url $SiteURL -UseWebLogin
    #To Create Folder and Files  
    if($MethodCall -eq "FOLDER")
    {
    	$FolderCnt=0
    	while($FolderCnt -lt $maxCount)
    	{
		    $FolderName= $FileName +"_"+ $FolderCnt
			write-host $FolderName
		    $SiteRelativePath=$SiteRelativeURL+"/"+$FolderName
			write-host $SiteRelativePath
		    Try
		    {
			    Add-PnPFolder -Name $FolderName -Folder $SiteRelativeURL -ErrorAction Stop
                Add-PnPFile -Folder $SiteRelativePath -Path $File
			   
		    }
		    catch 
		    {
    		    write-host "Folder Creation Error: $($_.Exception.Message)" -foregroundcolor Red
		    }
            $FolderCnt++
    	}
       
        write-output "New Folder and Files Created '$FolderName' Added! $($env:computername)" >> $LogFile 
         Write-host -f Green "Script execution completed...." |Out-File $LogFile -Append -Force 
          write-output "Script execution completed.... $($env:computername)" >> $LogFile -f Green
    }

    #To Create Files alone
    if($MethodCall -eq "FILE")
    {
        if($minorVersionCount -gt 0)
        {
            ensureLibraryIsUsingMinorVersions
        }
	    $FileCnt=0
	    while($FileCnt -lt $maxCount)
	    {
		    $NewFileName= $FileName+"_"+$FileCnt+".docx"
		    try
		    {
                for($i=0; $i -lt $minorVersionCount;$i++)
                {
                    if($i -gt 0 -and $i % $minorVersionCountBeforeMajor -eq 0)
                    {
                        $newfile = Add-PnPFile -Path $File -Folder $Folder -NewFileName $NewFileName
                        Set-PnPFileCheckedOut -Url $newfile.ServerRelativeUrl  
                        Set-PnPFileCheckedIn -Url $newfile.ServerRelativeUrl -CheckinType MajorCheckIn -Comment "Auto created" 
                    }
                    else
                    {
                        Add-PnPFile -Path $File -Folder $Folder -NewFileName $NewFileName
                    }
                    
                }
			    
		    }
		    catch
		    {
			    write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
		    }
		    $FileCnt++
		    Write-host -f Green "New File Created '$NewFileName' Added!" |Out-File $LogFile -Append -Force 
            write-output "New File Created '$NewFileName' Added! $($env:computername)" >> $LogFile -f Green
    
	    }
            Write-host -f Green "Script execution completed...." |Out-File $LogFile -Append -Force 
            write-output "Script execution completed.... $($env:computername)" >> $LogFile -f Green
    }
}
catch 
{
    write-host "Error: $($_.Exception.Message)" -foregroundcolor Red
}

 }
 else{
 write-host "Please type either File or Folder" -foregroundcolor Red
 }

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]


# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
# .\Create-DummyDocsWithVersions.ps1 -WebUrl "https://contoso.sharepoint.com/sites/Intranet" -ListTitle "Documents" -ServerRelativeUrl "/sites/intranet/Shared Documents" -ItemType File -ItemCount 5 -MajorVersions 3 -MinorVersionsPerMajor 2 -SourceFilePath "C:\\Temp\\Sample.docx"
[CmdletBinding(SupportsShouldProcess = $true)]
param (
  [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL hosting the library")]
  [ValidateNotNullOrEmpty()]
  [string]$WebUrl,
  [Parameter(Mandatory = $true, HelpMessage = "Display name of the target document library")]
  [ValidateNotNullOrEmpty()]
  [string]$ListTitle,
  [Parameter(Mandatory = $true, HelpMessage = "Server-relative URL of the folder that will host the generated items")]
  [ValidateNotNullOrEmpty()]
  [string]$ServerRelativeUrl,
  [Parameter(Mandatory = $true, HelpMessage = "Type of item to generate")]
  [ValidateSet('File','Folder', IgnoreCase = $true)]
  [string]$ItemType,
  [Parameter(Mandatory = $true, HelpMessage = "Number of items to create")]
  [ValidateRange(1, [int]::MaxValue)]
  [int]$ItemCount,
  [Parameter(Mandatory = $true, HelpMessage = "Number of major version cycles per item")]
  [ValidateRange(1, [int]::MaxValue)]
  [int]$MajorVersions,
  [Parameter(Mandatory = $true, HelpMessage = "Number of minor versions created before each major version")]
  [ValidateRange(1, [int]::MaxValue)]
  [int]$MinorVersionsPerMajor,
  [Parameter(Mandatory = $true, HelpMessage = "Local path to the template document used for uploads")]
  [ValidateNotNullOrEmpty()]
  [string]$SourceFilePath,
  [Parameter(HelpMessage = "Prefix applied to generated file and folder names")]
  [ValidateNotNullOrEmpty()]
  [string]$NamePrefix = 'Dummy'
)

begin {
  Write-Host "Ensuring CLI for Microsoft 365 session..." -ForegroundColor Cyan
  m365 login --ensure --output json | Out-Null
  Write-Host "Authenticated. Resolving library context." -ForegroundColor Green

  if (-not (Test-Path -LiteralPath $SourceFilePath -PathType Leaf)) {
    throw "Source file '$SourceFilePath' not found."
  }

  $listRaw = m365 spo list get --webUrl $WebUrl --title $ListTitle --output json 2>&1
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to retrieve list '$ListTitle'. CLI output: $listRaw"
  }

  $Script:List = $listRaw | ConvertFrom-Json

  if (-not $Script:List.EnableVersioning -or -not $Script:List.EnableMinorVersions) {
    Write-Host "Enabling major/minor versioning on '$ListTitle'." -ForegroundColor Yellow
    $listUpdate = m365 spo list set --webUrl $WebUrl --id $Script:List.Id --enableVersioning $true --enableMinorVersions $true --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
      throw "Failed to update library settings. CLI output: $listUpdate"
    }
  }

  $Script:TargetFolderUrl = $ServerRelativeUrl.TrimEnd('/')
  $Script:FileExtension = [System.IO.Path]::GetExtension($SourceFilePath)
  if ([string]::IsNullOrWhiteSpace($Script:FileExtension)) {
    $Script:FileExtension = '.dat'
  }

  $Script:Summary = [ordered]@{
    ItemsPlanned    = $ItemCount
    FoldersCreated  = 0
    FilesCreated    = 0
    VersionsCreated = 0
    Failures        = 0
  }

  Write-Host "Processing $ItemCount $ItemType item(s) inside '$ServerRelativeUrl'." -ForegroundColor Cyan
}

process {
  function Invoke-VersionAction {
    param (
      [Parameter(Mandatory = $true)]
      [string]$FileUrl,
      [Parameter(Mandatory = $true)]
      [ValidateSet('Minor','Major')]
      [string]$Type,
      [string]$Comment = 'Auto-generated version'
    )

    if (-not $PSCmdlet.ShouldProcess($FileUrl, "Create $Type version")) {
      return $true
    }

    $checkout = m365 spo file checkout --webUrl $WebUrl --url $FileUrl --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
      $Script:Summary.Failures++
      Write-Warning "Failed to check out '$FileUrl'. CLI output: $checkout"
      return $false
    }

    $checkin = m365 spo file checkin --webUrl $WebUrl --url $FileUrl --type $Type --comment $Comment --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
      $Script:Summary.Failures++
      Write-Warning "Failed to check in $Type version for '$FileUrl'. CLI output: $checkin"
      return $false
    }

    $Script:Summary.VersionsCreated++
    return $true
  }

  function Invoke-Upload {
    param (
      [Parameter(Mandatory = $true)]
      [string]$DestinationFolder,
      [Parameter(Mandatory = $true)]
      [string]$LeafName,
      [Parameter(Mandatory = $true)]
      [string]$FileUrl
    )

    if (-not $PSCmdlet.ShouldProcess($FileUrl, 'Upload file')) {
      return $true
    }

    $upload = m365 spo file add --webUrl $WebUrl --folder $DestinationFolder --path $SourceFilePath --FileLeafRef $LeafName --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
      $Script:Summary.Failures++
      Write-Warning "Failed to upload '$LeafName'. CLI output: $upload"
      return $false
    }

    return $true
  }

  for ($index = 1; $index -le $ItemCount; $index++) {
    $progressPercent = [math]::Round(($index / $ItemCount) * 100, 2)
    Write-Progress -Activity "Creating items" -Status "Processing item $index of $ItemCount" -PercentComplete $progressPercent

    $baseName = "{0}-{1}" -f $NamePrefix, $index
    $destination = $Script:TargetFolderUrl

    if ($ItemType.Equals('Folder', 'InvariantCultureIgnoreCase')) {
      $destination = "$($Script:TargetFolderUrl)/$baseName"
      if ($PSCmdlet.ShouldProcess($destination, 'Create folder')) {
        $folder = m365 spo folder add --webUrl $WebUrl --parentFolderUrl $Script:TargetFolderUrl --name $baseName --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
          $Script:Summary.Failures++
          Write-Warning "Failed to create folder '$baseName'. CLI output: $folder"
          continue
        }
        $Script:Summary.FoldersCreated++
      }
      else {
        $Script:Summary.FoldersCreated++
      }
    }

    $leafName = "$baseName$($Script:FileExtension)"
    $fileUrl = "$destination/$leafName"

    if (-not (Invoke-Upload -DestinationFolder $destination -LeafName $leafName -FileUrl $fileUrl)) {
      continue
    }

    $Script:Summary.FilesCreated++

    if (-not (Invoke-VersionAction -FileUrl $fileUrl -Type Major -Comment 'Initial major version')) {
      continue
    }

    for ($major = 1; $major -le $MajorVersions; $major++) {
      $minorFailed = $false

      for ($minor = 1; $minor -le $MinorVersionsPerMajor; $minor++) {
        if (-not (Invoke-VersionAction -FileUrl $fileUrl -Type Minor -Comment "Minor version $minor for major cycle $major")) {
          $minorFailed = $true
          break
        }
      }

      if ($minorFailed) {
        break
      }

      if (-not (Invoke-VersionAction -FileUrl $fileUrl -Type Major -Comment "Major version $major")) {
        break
      }
    }
  }
}

end {
  Write-Host "========== Summary ==========" -ForegroundColor Cyan
  Write-Host "Items planned    : $($Script:Summary.ItemsPlanned)"
  Write-Host "Folders created  : $($Script:Summary.FoldersCreated)"
  Write-Host "Files created    : $($Script:Summary.FilesCreated)"
  Write-Host "Versions created : $($Script:Summary.VersionsCreated)"
  Write-Host "Failures         : $($Script:Summary.Failures)"
  Write-Host "=============================" -ForegroundColor Cyan
}
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| Kasper Larsen|
| Mathijs Verbeeck|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/create-dummy-docs-versions-in-library" aria-hidden="true" />

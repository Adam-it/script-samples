

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
# Usage example:
#   .\Create-Bulk-Dummy-Documents.ps1 -WebUrl "https://contoso.sharepoint.com/sites/Intranet" -ListTitle "Documents" -ServerRelativeUrl "/sites/intranet/Shared Documents" -Type File -ItemsToCreate 5 -MajorVersions 6 -MinorVersionBeforeMajor 3 -FileToUse "D:\Temp\DummyFile.docx"
#   .\Create-Bulk-Dummy-Documents.ps1 -WebUrl "https://contoso.sharepoint.com/sites/Intranet" -ListTitle "Documents" -ServerRelativeUrl "/sites/intranet/Shared Documents" -Type Folder -ItemsToCreate 5 -MajorVersions 6 -MinorVersionBeforeMajor 3 -FileToUse "D:\Temp\DummyFile.docx"
[CmdletBinding()]
param (
  [Parameter(Mandatory = $true, HelpMessage = "Web url from which to create the files, e.g. https://contoso.sharepoint.com/sites/Intranet")]
  [string]$WebUrl,
  [Parameter(Mandatory = $true, HelpMessage = "Title of the list on which to create the documents or files")]
  [string]$ListTitle,
  [Parameter(Mandatory = $true, HelpMessage = "Server relative URL of the folder in which to create the documents")]
  [string]$ServerRelativeUrl,
  [Parameter(Mandatory = $true, HelpMessage = "Type of content to create, possible values are 'file' or 'folder'")]
  [ValidateSet("File","Folder","file","folder")]
  [string]$Type,
  [Parameter(Mandatory = $true, HelpMessage = "Amount of items to create")]
  [int]$ItemsToCreate,
  [Parameter(Mandatory = $true, HelpMessage = "Amount of major versions to create")]
  [int]$MajorVersions,
  [Parameter(Mandatory = $true, HelpMessage = "This will define the amount of minor versions that will be created before a major version is added")]
  [int]$MinorVersionBeforeMajor,
  [Parameter(Mandatory = $true, HelpMessage = "Path of the file to use when creating versions")]
  [string]$FileToUse
)
begin {
  $script:Stats = [ordered]@{
    ItemsProcessed  = 0
    ItemsSucceeded  = 0
    VersionFailures = 0
    CreationFailures = 0
  }

  function Invoke-CheckoutCheckin {
    param (
      [Parameter(Mandatory = $true)]
      [string]$FileUrl,
      [Parameter(Mandatory = $true)]
      [ValidateSet('Major', 'Minor')]
      [string]$Type,
      [Parameter(Mandatory = $false)]
      [string]$Comment
    )

    $operations = @(
      @{ Args = @('spo', 'file', 'checkout', '--webUrl', $WebUrl, '--fileUrl', $FileUrl); Message = "Failed to check out file '$FileUrl' before $Type check-in." },
      @{ Args = @('spo', 'file', 'checkin', '--webUrl', $WebUrl, '--fileUrl', $FileUrl, '--type', $Type); Message = "Failed to check in $Type version for file '$FileUrl'." }
    )

    if (-not [string]::IsNullOrWhiteSpace($Comment)) {
      $operations[1].Args += @('--comment', $Comment)
    }

    foreach ($operation in $operations) {
      $output = m365 @($operation.Args) 2>&1
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "$($operation.Message) CLI output: $output"
        return $false
      }
    }

    return $true
  }

  function New-Versions {
    param (
      [Parameter(Mandatory = $true)]
      [string]$FileUrl,
      [Parameter(Mandatory = $true)]
      [int]$Counter
    )

    $script:Stats.ItemsProcessed++
  
    # Have to first check out, else it throws an error 'Error: The file "Shared Documents/1.docx" is not checked out'
    if (-not (Invoke-CheckoutCheckin -FileUrl $FileUrl -Type 'Major' -Comment 'First major version check in')) {
      $script:Stats.VersionFailures++
      return $false
    }
    
    # Creating versions
    for ($i = 1; $i -lt ($MajorVersions + 1); $i++) {
      Write-Progress -Activity "Creating versions" -Status "$Counter/$ItemsToCreate files created. Creating major version $i/$MajorVersions" -PercentComplete (($Counter / $ItemsToCreate) * 100)
      for ($j = 1; $j -lt ($MinorVersionBeforeMajor + 1); $j++) {
        Write-Progress -Activity "Creating versions" -Status "$Counter/$ItemsToCreate files created. Processing major version $i/$MajorVersions, Creating minor version $j/$MinorVersionBeforeMajor" -PercentComplete (($Counter / $ItemsToCreate) * 100)
        # Create minor version here
        if (-not (Invoke-CheckoutCheckin -FileUrl $FileUrl -Type 'Minor' -Comment "Check in of minor version $j")) {
          $script:Stats.VersionFailures++
          return $false
        }
      }
      if (-not (Invoke-CheckoutCheckin -FileUrl $FileUrl -Type 'Major' -Comment "Check in of major version $i")) {
        $script:Stats.VersionFailures++
        return $false
      }
    }

    return $true
  }

  $loginOutput = m365 login --ensure 2>&1
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to authenticate to Microsoft 365. CLI output: $loginOutput"
  }
  Write-Host "Initialization done, time to create versions!" -f Green 
}
process {
  $FileExtension = $FileToUse.Split('.')[$FileToUse.Split('.').Count - 1]

  Write-Host "Starting the script, we are going to create $ItemsToCreate $($type)s in the list with title $ListTitle. They will each have $MajorVersions major versions and $MinorVersionBeforeMajor minor versions before a new major version is added." -f Green
  $listOutput = m365 spo list get --webUrl $WebUrl --title $ListTitle --output json 2>&1
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to retrieve list '$ListTitle'. CLI output: $listOutput"
  }

  try {
    $list = $listOutput | ConvertFrom-Json
  }
  catch {
    throw "Unable to parse list details. Error: $($_.Exception.Message)"
  }
  Write-Host "Obtained the list with title $($list.Title)" -f Green

  if (!$list.EnableMinorVersions) {
    Write-Host "Have to set properties on list to enable versioning and enable the creation of minor versions" -f Red
    $listSetOutput = m365 spo list set --webUrl $WebUrl --id $list.Id --enableVersioning $true --enableMinorVersions $true 2>&1
    if ($LASTEXITCODE -ne 0) {
      throw "Failed to update list properties for '$ListTitle'. CLI output: $listSetOutput"
    }
    Write-Host "List properties updated!" -f Green
  }

  Write-Host "Obtained the folder with server relative url $ServerRelativeUrl" -f Green
  Write-Host "Time to start the creation process..." -f Green
  if ($Type.ToLower() -eq "folder") {
    $FolderCnt = 1
    while ($FolderCnt -le $ItemsToCreate) {
      Write-Progress -Activity "Creating folders" -Status "$FolderCnt/$ItemsToCreate folders created" -PercentComplete (($FolderCnt / $ItemsToCreate) * 100)
   
      $FileUrl = "$($ServerRelativeUrl)/$FolderCnt/$FolderCnt.$FileExtension" 

      # Create the folder and add the file
      $folderAddOutput = m365 spo folder add --webUrl $WebUrl --parentFolderUrl $ServerRelativeUrl --name $FolderCnt 2>&1
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to create folder '$FolderCnt'. CLI output: $folderAddOutput"
        $script:Stats.CreationFailures++
        $FolderCnt++
        continue
      }

      $fileAddOutput = m365 spo file add --webUrl $WebUrl --folder "$($ServerRelativeUrl)/$FolderCnt" --path $FileToUse --fileName $FolderCnt 2>&1
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to upload file '$FolderCnt'. CLI output: $fileAddOutput"
        $script:Stats.CreationFailures++
        $FolderCnt++
        continue
      }

      # Call function to create the versions
      if (-not (New-Versions -FileUrl $FileUrl -Counter $FolderCnt)) {
        Write-Warning "Failed to create versions for item '$FolderCnt'. Skipping to next item."
        $FolderCnt++
        continue
      }

      $script:Stats.ItemsSucceeded++

      $FolderCnt++
    }
  } elseif ($Type.ToLower() -eq "file") {
    $FileCnt = 1
    while ($FileCnt -le $ItemsToCreate) {
      Write-Progress -Activity "Creating files" -Status "$FileCnt/$ItemsToCreate files created" -PercentComplete (($FileCnt / $ItemsToCreate) * 100)
    
      $FileUrl = "$($ServerRelativeUrl)/$FileCnt.$FileExtension" 

      # Creating file
      $fileAddOutput = m365 spo file add --webUrl $WebUrl --folder $ServerRelativeUrl --path $FileToUse --fileName $FileCnt 2>&1
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to upload file '$FileCnt'. CLI output: $fileAddOutput"
        $script:Stats.CreationFailures++
        $FileCnt++
        continue
      }
      
      # Call function to create the versions
      if (-not (New-Versions -FileUrl $FileUrl -Counter $FileCnt)) {
        Write-Warning "Failed to create versions for item '$FileCnt'. Skipping to next item."
        $FileCnt++
        continue
      }

      $script:Stats.ItemsSucceeded++

      $FileCnt++
    }
  }

  Write-Host "Script Complete! :)" -f Green
}

end {
  Write-Host "Summary" -ForegroundColor Green
  Write-Host "Items processed : $($script:Stats.ItemsProcessed)"
  Write-Host "Items succeeded : $($script:Stats.ItemsSucceeded)"
  Write-Host "Creation failures: $($script:Stats.CreationFailures)"
  Write-Host "Version failures : $($script:Stats.VersionFailures)"
}

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

## Contributors

| Author(s) |
|-----------|
| Kasper Larsen|
| Mathijs Verbeeck|
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/create-dummy-docs-versions-in-library" aria-hidden="true" />

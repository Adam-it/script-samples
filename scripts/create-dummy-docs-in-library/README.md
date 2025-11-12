

# Create bulk dummy documents in SharePoint Document library

## Summary

There are times when we have to replicate scenario to bulk upload dummy documents in large numbers for replicating 5000 items limit or testing performance of dec/test/uat enviorments. This script would help us create 'n' number of dummy documents specified as maxCount in script. Script will also provide option to create dummy folder first for each file and then upload file inside that folder. Script will use the specified file and add counter inside file name to provide uniqueness of file.

Note about two available options
- Upload the dummy files directly on the SP library, you can provide this path in "$Folder"
- Create a dummy folder first and upload the file inside that folder, you can provide the root path in "$SiteRelativeURL"


## Implementation

- Open Windows PowerShell ISE
- Create a new file
- Write a script as below,
- Change the variables to target to your enviorment, site, document library, document path, max count
- Run the script.
 
## Screenshot of Output 

 Below is the output after I have ran the script twice with maxCount set to 5, 

- Input as Folder (it has created five folder with auto incrementing folder name to get uniqueness and then added file inside each folder)
- Input as File  (it has created five files and auto incremented file name to get uniqueness)

![Example Screenshot](assets/preview.png)

# [PnP PowerShell](#tab/pnpps)
```powershell

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
$maxCount = 5


#For Sample Document Creation the file needs to be part of some location.
$FilePath= Get-ChildItem "D:\SP\repos\myscriptsamples\Dummy.docx"
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
			write-host $SiteRelativePath

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
	    $FileCnt=0
	    while($FileCnt -lt $maxCount)
	    {
		    $NewFileName= $FileName+"_"+$FileCnt+".docx"
		    try
		    {
			    Add-PnPFile -Path $File -Folder $Folder -NewFileName $NewFileName
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
***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding(SupportsShouldProcess = $true)]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site that hosts the library.")]
    [string]$SiteUrl,

    [Parameter(Mandatory, HelpMessage = "Server- or site-relative URL of the target library or folder.")]
    [string]$TargetFolderUrl,

    [Parameter(Mandatory, HelpMessage = "Local path to the seed file that will be duplicated.")]
    [string]$SourceFilePath,

    [Parameter(Mandatory, HelpMessage = "Number of dummy files (or folders) to create.")]
    [ValidateRange(1, [int]::MaxValue)]
    [int]$ItemCount,

    [Parameter(HelpMessage = "When provided, a folder will be created per item before uploading the file.")]
    [switch]$CreateFolders
)

begin {
    if (-not (Test-Path -Path $SourceFilePath -PathType Leaf)) {
        throw "Seed file not found at path: $SourceFilePath"
    }

    m365 login --ensure

    $script:Summary = [ordered]@{
        ItemsRequested  = $ItemCount
        FoldersCreated  = 0
        FilesUploaded   = 0
        ItemsSimulated  = 0
        Failures        = 0
    }

    $script:SeedFile = Get-Item -Path $SourceFilePath
    $script:SeedBase = [System.IO.Path]::GetFileNameWithoutExtension($SeedFile.Name)
    $script:SeedExt  = $SeedFile.Extension

    Write-Host "Preparing to create $ItemCount item(s) in '$TargetFolderUrl'"
    if ($CreateFolders) {
        Write-Host "Each item will reside in its own folder."
    }
}

process {
    for ($index = 1; $index -le $ItemCount; $index++) {
        $safeIndex = '{0:D4}' -f $index
        $folderUrl = $TargetFolderUrl

        if ($CreateFolders) {
            $folderName = "$SeedBase-$safeIndex"
            $folderUrl = if ($TargetFolderUrl.StartsWith('/')) {
                "$TargetFolderUrl/$folderName"
            } else {
                "$TargetFolderUrl/$folderName"
            }

            if ($PSCmdlet.ShouldProcess($folderUrl, 'Create folder')) {
                $folderResult = m365 spo folder add --webUrl $SiteUrl --parentFolderUrl $TargetFolderUrl --name $folderName --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to create folder '$folderName'. CLI: $folderResult"
                    $Summary.Failures++
                    continue
                }
                $Summary.FoldersCreated++
            } else {
                Write-Host "  WhatIf: folder creation skipped."
            }
        }

        $newFileName = "$SeedBase-$safeIndex$SeedExt"
        $targetDescription = "$folderUrl/$newFileName"

        if ($PSCmdlet.ShouldProcess($targetDescription, 'Upload file')) {
            $tempFile = Join-Path -Path ([System.IO.Path]::GetTempPath()) -ChildPath $newFileName
            try {
                Copy-Item -Path $SeedFile.FullName -Destination $tempFile -Force

                $uploadResult = m365 spo file add --webUrl $SiteUrl --folder $folderUrl --path $tempFile --output json 2>&1
                if ($LASTEXITCODE -ne 0) {
                    Write-Warning "Failed to upload file '$newFileName'. CLI: $uploadResult"
                    $Summary.Failures++
                    continue
                }

                $Summary.FilesUploaded++
            }
            finally {
                Remove-Item -Path $tempFile -ErrorAction SilentlyContinue
            }
        } else {
            Write-Host "  WhatIf: upload skipped."
            $Summary.ItemsSimulated++
        }
    }
}

end {
    Write-Host "Creation summary:" -ForegroundColor Cyan
    Write-Host ("  Items requested : {0}" -f $Summary.ItemsRequested)
    if ($CreateFolders) {
        Write-Host ("  Folders created : {0}" -f $Summary.FoldersCreated)
    }
    Write-Host ("  Files uploaded  : {0}" -f $Summary.FilesUploaded)
    if ($Summary.ItemsSimulated -gt 0) {
        Write-Host ("  Items simulated : {0}" -f $Summary.ItemsSimulated)
    }
    Write-Host ("  Failures        : {0}" -f $Summary.Failures)
}
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***


## Contributors

| Author(s) |
|-----------|
| Siddharth Vaghasia|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/create-dummy-docs-in-library" aria-hidden="true" />

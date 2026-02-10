

# Create and add site design to SharePoint site with site columns, content type. 

## Summary

This script creates and applies a site design with custom column types to a SharePoint site. This sample has been modernized to use CLI for Microsoft 365 v11.4.0+ and follows modern PowerShell best practices including typed parameters, WhatIf support, error handling, and progress reporting.

The script:
 - Sets regional settings
 - Creates custom content type 
 - Adds new fields to new content type
 - Creates new list with new content type
 - Changes list view to custom one

More about site design schema
 [https://learn.microsoft.com/sharepoint/dev/declarative-customization/site-design-json-schema](https://learn.microsoft.com/sharepoint/dev/declarative-customization/site-design-json-schema)

![Example Screenshot](assets/example.png)

# [PnP PowerShell](#tab/pnpps)

```powershell

###### Declare and Initialize Variables ######  

#Destination site collection url
$url="https://<tenant>.sharepoint.com/sites/sitename"

# log file will be saved in same directory script was started from  
$currentTime= $(get-date).ToString("yyyyMMddHHmmss")  
$logFilePath=".\log-"+$currentTime+".log"  



## Start the Transcript  
Start-Transcript -Path $logFilePath 

## Connect to SharePoint Online site  
Connect-PnPOnline -Url $Url -Interactive


#Site design script
# - Set site regionalsettings (useful for formatting date type field)
# - Apply custom theme
# - Create site columns (text, number, person, choice)
# - Create site content type with created site columns

#site design version, can be static, for me it is easier check versions
$v = "1"

#content type Id
$ctId = "0x010100A45633E36EDA6040B00F4AE79CBF8F32"

$site_script = '{
    "$schema": "https://developer.microsoft.com/json-schemas/sp/site-design-script-actions.schema.json",
    "actions": [
         {
          "verb": "setRegionalSettings",
          "timeZone": 4, 
          "locale": 2057, 
          "sortOrder": 25, 
          "hourFormat": "24"
        },
        {
        "verb": "applyTheme",
        "themeJson": {
          "version": "2",
          "isInverted": false,
          "palette": {
            "themePrimary":"#0047ba",
            "themeLighterAlt":"#f2f6fc",
            "themeLighter":"#c3d4eb",
            "themeLight":"#7aabde",
            "themeTertiary":"#0091a5",
            "themeSecondary":"#c3d4eb",
            "themeDarkAlt":"#0040a8",
            "themeDark":"#00368d",
            "themeDarker":"#002868",
            "neutralLighterAlt":"#f8f8f8",
            "neutralLighter":"#f4f4f4",
            "neutralLight":"#eaeaea",
            "neutralQuaternaryAlt":"#dadada",
            "neutralQuaternary":"#d0d0d0",
            "neutralTertiaryAlt":"#c8c8c8",
            "neutralTertiary":"#595959",
            "neutralSecondary":"#373737",
            "neutralPrimaryAlt":"#2f2f2f",
            "neutralPrimary":"#000000",
            "neutralDark":"#151515",
            "black":"#0b0b0b",
            "white":"#ffffff",
            "primaryBackground":"#ffffff",
            "primaryText":"#000000",
            "bodyBackground":"#ffffff",
            "bodyText":"#000000",
            "disabledBackground":"#f4f4f4",
            "disabledText":"#c8c8c8"
           }
         }
        },
        {
          "verb": "createSiteColumnXml",
          "schemaXml": "<Field ID=\"{e1605fb4-611a-4eae-b119-77b4b633b436}\" Type=\"MultiChoice\" DisplayName=\"Hobbies\" Required=\"FALSE\" FillInChoice=\"FALSE\" StaticName=\"Hobbies\" Name=\"Hobbies\"><DefaultFormula>=\"\"</DefaultFormula><CHOICES><CHOICE>PingPong</CHOICE><CHOICE>Books</CHOICE><CHOICE>Sharing Is Carring</CHOICE><CHOICE>This is easter egg</CHOICE><CHOICE>Wearing Mask</CHOICE><CHOICE>Face Tatoos</CHOICE><CHOICE>Flying with no Cape</CHOICE></CHOICES></Field>"
        },
        {
          "verb": "createSiteColumn",
          "fieldType": "Text",
          "internalName": "SimpleTextField",
          "displayName": "SimpleTextField",
          "isRequired": false,
          "id": "60f37466-fd00-47a3-9665-44b48a743e27"
        },
        {
          "verb": "createSiteColumn",
          "fieldType": "User",
          "internalName": "siteColumn4User",
          "displayName": "Owner",
          "isRequired": false,
          "id": "181c4370-cdae-471b-9499-730046e55b75"
        },
        {
          "verb": "createSiteColumn",
          "fieldType": "Number",
          "internalName": "Number",
          "displayName": "Number",
          "isRequired": false,
          "id": "151c4370-cdae-471b-9499-730046e55b78"
        },
        {
          "verb": "createContentType",
          "name": "Powershel Samples",
          "description": "Create something with fields",
          "id":"'+$ctId+'",
          "hidden": false,
          "group": "Power Samples",
          "subactions":
            [
              {
                "verb": "addSiteColumn",
                "internalName": "Hobbies"
              },
              {
                "verb": "addSiteColumn",
                "internalName": "SimpleTextField"
              },
              {
                "verb": "addSiteColumn",
                "internalName": "siteColumn4User"
              },
              {
                "verb": "addSiteColumn",
                "internalName": "Number"
              }
            ]
        }
 ],
    "bindata": { },
    "version": "'+$v+'"
}'

#site script to create list and update it with columns by adding content type
 $site_script_CreateAndUpdateSiteList = '
 {
    "$schema": "https://developer.microsoft.com/json-schemas/sp/site-design-script-actions.schema.json",
    "actions": [
    {
                  "verb": "createSPList",
                  "listName": "DemoListForPowerShellSamples",
                  "templateType": 100,
                  "subactions": [
                  {
                      "verb": "addContentType",
                      "name": "Powershel Samples"
                    },
                    {
                        "verb": "addSPView",
                        "name": "Custom View",
                        "viewFields":
                        [
                          "Hobbies",
                          "Number"
                        ],
                        "query": "<OrderBy><FieldRef Title=\"FileLeafRef\" Ascending=\"TRUE\" /></OrderBy>",
                        "rowLimit": 100,
                        "isPaged": true,
                        "makeDefault": true
                    },
                    {
                        "verb": "removeContentType",
                        "name": "Item"
                    }
                  
                  ]
                  
    }
            
],
     "bindata": { },
    "version": "'+$v+'"
}'

 #add Script to SharePoint sharepoint tenant
 $addScript = Add-PnPSiteScript -Title "This is first script"  -Content $site_script  -Description "Sets regional settings and creates site columns and content type."
 $site_script_CreateAndUpdateSiteList = Add-PnPSiteScript  -Title "This is second script for list"  -Content $site_script_CreateAndUpdateSiteList  -Description "Create and Update list"

 #add site design to site collection with site script
 $siteDesign = Add-PnPSiteDesign  -Title "DevGods site design"  -WebTemplate "64"  -SiteScriptIds  $addScript.Id,  $site_script_CreateAndUpdateSiteList.Id -Description "Site design for the sample"

 #set design on site collection
 Set-PnPSiteDesign -Identity $siteDesign.Id -Title "DevGods site design"  -WebTemplate "64"  -SiteScriptIds  $addScript.Id,  $site_script_CreateAndUpdateSiteList.Id -Description "Site design for the sample"

 #invoke site design
 Invoke-PnPSiteDesign -Identity $siteDesign.Id -WebUrl $url


## Disconnect the context  
Disconnect-PnPOnline  
 
## Stop Transcript  
Stop-Transcript  

```

> [!Note]
> SharePoint tenant admin right are required to be able add site design

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
[CmdletBinding(SupportsShouldProcess)]
param(
    [Parameter(Mandatory, HelpMessage = "URL of the SharePoint site where the site design will be applied")]
    [ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)/.+')]
    [string]$SiteUrl,

    [Parameter(Mandatory, HelpMessage = "Path to the first JSON site script file (site columns, content types, theme)")]
    [ValidateNotNullOrEmpty()]
    [string]$FirstScriptPath,

    [Parameter(Mandatory, HelpMessage = "Path to the second JSON site script file (list creation, views)")]
    [ValidateNotNullOrEmpty()]
    [string]$SecondScriptPath,

    [Parameter(Mandatory, HelpMessage = "Title for the site design")]
    [ValidateNotNullOrEmpty()]
    [string]$SiteDesignTitle,

    [Parameter(HelpMessage = "Description for the site design")]
    [string]$SiteDesignDescription = "Site design created with CLI for Microsoft 365",

    [Parameter(HelpMessage = "Web template type for the site design")]
    [ValidateSet('TeamSite', 'CommunicationSite')]
    [string]$WebTemplate = 'TeamSite',

    [Parameter(HelpMessage = "Version number for the site design")]
    [int]$SiteDesignVersion = 1,

    [Parameter(HelpMessage = "Directory path where the transcript log will be saved. Defaults to current directory")]
    [string]$OutputPath = (Get-Location).Path
)

begin {
    if ($PSBoundParameters.ContainsKey('OutputPath')) {
        if (-not (Test-Path -Path $OutputPath -PathType Container)) {
            throw "Output path '$OutputPath' does not exist or is not a directory."
        }
    }

    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $transcriptPath = Join-Path -Path $OutputPath -ChildPath "SiteDesign-Transcript-$timestamp.log"
    Start-Transcript -Path $transcriptPath

    Write-Verbose "Starting site design creation and application process..."

    $script:Summary = @{
        ScriptsCreated = 0
        DesignsCreated = 0
        DesignsApplied = 0
        Failures = 0
    }

    try {
        Write-Verbose "Validating JSON script file paths..."
        
        if (-not (Test-Path -Path $FirstScriptPath -PathType Leaf)) {
            throw "First script file not found at path: $FirstScriptPath"
        }
        
        if (-not (Test-Path -Path $SecondScriptPath -PathType Leaf)) {
            throw "Second script file not found at path: $SecondScriptPath"
        }

        Write-Verbose "Validating JSON content..."
        try {
            $null = Get-Content -Path $FirstScriptPath -Raw | ConvertFrom-Json -ErrorAction Stop
            $null = Get-Content -Path $SecondScriptPath -Raw | ConvertFrom-Json -ErrorAction Stop
        }
        catch {
            throw "Invalid JSON in script files: $_"
        }

        Write-Progress -Activity "Site Design Setup" -Status "Authenticating to Microsoft 365" -PercentComplete 5
        Write-Verbose "Ensuring user is logged in to Microsoft 365..."
        m365 login --ensure
        
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to authenticate to Microsoft 365. Please check your credentials and try again."
        }
        
        Write-Verbose "Successfully authenticated to Microsoft 365"

        Write-Progress -Activity "Site Design Setup" -Status "Validating site URL" -PercentComplete 10
        Write-Verbose "Retrieving site information for $SiteUrl..."
        $siteJson = m365 spo site get --url $SiteUrl --output json
        
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve site information for $SiteUrl. Ensure the URL is correct and you have access."
        }
        
        $site = $siteJson | ConvertFrom-Json
        Write-Verbose "Successfully validated site: $($site.Title)"
    }
    catch {
        Write-Error $_
        Stop-Transcript
        throw
    }
}

process {
    try {
        Write-Progress -Activity "Site Design Setup" -Status "Creating first site script (columns, content types, theme)" -PercentComplete 20
        
        $firstScriptContent = Get-Content -Path $FirstScriptPath -Raw
        $firstScriptTitle = "$SiteDesignTitle - Script 1 (Site Columns & Content Types)"
        
        if ($PSCmdlet.ShouldProcess($firstScriptTitle, 'Create site script')) {
            Write-Verbose "Creating first site script: $firstScriptTitle"
            $firstScriptJson = m365 spo sitescript add --title $firstScriptTitle --description "Site columns, content types, and theme configuration" --content $firstScriptContent --output json
            
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to create first site script. Skipping remaining operations."
                $script:Summary.Failures++
                return
            }
            
            $firstScript = $firstScriptJson | ConvertFrom-Json
            $script:Summary.ScriptsCreated++
            Write-Host "Created site script: $firstScriptTitle (ID: $($firstScript.Id))" -ForegroundColor Green
        }

        Write-Progress -Activity "Site Design Setup" -Status "Creating second site script (list, views)" -PercentComplete 40
        
        $secondScriptContent = Get-Content -Path $SecondScriptPath -Raw
        $secondScriptTitle = "$SiteDesignTitle - Script 2 (List & Views)"
        
        if ($PSCmdlet.ShouldProcess($secondScriptTitle, 'Create site script')) {
            Write-Verbose "Creating second site script: $secondScriptTitle"
            $secondScriptJson = m365 spo sitescript add --title $secondScriptTitle --description "List creation and view configuration" --content $secondScriptContent --output json
            
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to create second site script. Skipping remaining operations."
                $script:Summary.Failures++
                return
            }
            
            $secondScript = $secondScriptJson | ConvertFrom-Json
            $script:Summary.ScriptsCreated++
            Write-Host "Created site script: $secondScriptTitle (ID: $($secondScript.Id))" -ForegroundColor Green
        }

        Write-Progress -Activity "Site Design Setup" -Status "Creating site design" -PercentComplete 60
        
        if ($PSCmdlet.ShouldProcess($SiteDesignTitle, 'Create site design')) {
            Write-Verbose "Creating site design: $SiteDesignTitle"
            $siteDesignJson = m365 spo sitedesign add --title $SiteDesignTitle --description $SiteDesignDescription --webTemplate $WebTemplate --siteScripts "$($firstScript.Id),$($secondScript.Id)" --output json
            
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to create site design. Skipping remaining operations."
                $script:Summary.Failures++
                return
            }
            
            $siteDesign = $siteDesignJson | ConvertFrom-Json
            $script:Summary.DesignsCreated++
            Write-Host "Created site design: $SiteDesignTitle (ID: $($siteDesign.Id))" -ForegroundColor Green
        }

        Write-Progress -Activity "Site Design Setup" -Status "Updating site design version" -PercentComplete 75
        
        if ($PSCmdlet.ShouldProcess("$SiteDesignTitle (Version $SiteDesignVersion)", 'Update site design version')) {
            Write-Verbose "Updating site design version to $SiteDesignVersion..."
            $null = m365 spo sitedesign set --id $siteDesign.Id --version $SiteDesignVersion --output json
            
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to update site design version."
                $script:Summary.Failures++
            }
            else {
                Write-Verbose "Site design version updated to $SiteDesignVersion"
            }
        }

        Write-Progress -Activity "Site Design Setup" -Status "Applying site design to site" -PercentComplete 90
        
        if ($PSCmdlet.ShouldProcess($SiteUrl, "Apply site design '$SiteDesignTitle'")) {
            Write-Verbose "Applying site design to $SiteUrl..."
            $applyResultJson = m365 spo sitedesign apply --id $siteDesign.Id --webUrl $SiteUrl --output json
            
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to apply site design to site."
                $script:Summary.Failures++
            }
            else {
                $script:Summary.DesignsApplied++
                Write-Host "Successfully applied site design to $SiteUrl" -ForegroundColor Green
            }
        }

        Write-Progress -Activity "Site Design Setup" -Status "Completed" -PercentComplete 100 -Completed
    }
    catch {
        Write-Warning "Error during site design setup: $_"
        $script:Summary.Failures++
    }
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Site Design Setup Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Site Scripts Created: " -NoNewline
    Write-Host $script:Summary.ScriptsCreated -ForegroundColor $(if ($script:Summary.ScriptsCreated -gt 0) { 'Green' } else { 'Yellow' })
    Write-Host "Site Designs Created: " -NoNewline
    Write-Host $script:Summary.DesignsCreated -ForegroundColor $(if ($script:Summary.DesignsCreated -gt 0) { 'Green' } else { 'Yellow' })
    Write-Host "Site Designs Applied: " -NoNewline
    Write-Host $script:Summary.DesignsApplied -ForegroundColor $(if ($script:Summary.DesignsApplied -gt 0) { 'Green' } else { 'Yellow' })
    Write-Host "Failures: " -NoNewline
    Write-Host $script:Summary.Failures -ForegroundColor $(if ($script:Summary.Failures -eq 0) { 'Green' } else { 'Red' })
    Write-Host "========================================`n" -ForegroundColor Cyan

    Write-Verbose "Transcript saved to: $transcriptPath"
    Stop-Transcript
}

# Example 1: Basic usage with site design for a Team site
# .\Create-SiteDesignWithCustomList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/demo" -FirstScriptPath ".\firstscript.json" -SecondScriptPath ".\secondscript.json" -SiteDesignTitle "Contoso Site Design"

# Example 2: Use WhatIf to preview actions without making changes
# .\Create-SiteDesignWithCustomList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/demo" -FirstScriptPath ".\firstscript.json" -SecondScriptPath ".\secondscript.json" -SiteDesignTitle "Contoso Site Design" -WhatIf

# Example 3: Create site design for Communication site with custom version
# .\Create-SiteDesignWithCustomList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/comms" -FirstScriptPath ".\firstscript.json" -SecondScriptPath ".\secondscript.json" -SiteDesignTitle "Comms Site Design" -WebTemplate "CommunicationSite" -SiteDesignVersion 2

# Example 4: Verbose output with custom transcript path
# .\Create-SiteDesignWithCustomList.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/demo" -FirstScriptPath ".\firstscript.json" -SecondScriptPath ".\secondscript.json" -SiteDesignTitle "Contoso Site Design" -OutputPath "C:\Logs" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]


# [JSON Site Script](#tab/json1)

```
{
    "$schema": "https://developer.microsoft.com/json-schemas/sp/site-design-script-actions.schema.json",
    "actions": [
         {
          "verb": "setRegionalSettings",
          "timeZone": 4, 
          "locale": 2057, 
          "sortOrder": 25, 
          "hourFormat": "24"
        },
        {
        "verb": "applyTheme",
        "themeJson": {
          "version": "2",
          "isInverted": false,
          "palette": {
            "themePrimary":"#0047ba",
            "themeLighterAlt":"#f2f6fc",
            "themeLighter":"#c3d4eb",
            "themeLight":"#7aabde",
            "themeTertiary":"#0091a5",
            "themeSecondary":"#c3d4eb",
            "themeDarkAlt":"#0040a8",
            "themeDark":"#00368d",
            "themeDarker":"#002868",
            "neutralLighterAlt":"#f8f8f8",
            "neutralLighter":"#f4f4f4",
            "neutralLight":"#eaeaea",
            "neutralQuaternaryAlt":"#dadada",
            "neutralQuaternary":"#d0d0d0",
            "neutralTertiaryAlt":"#c8c8c8",
            "neutralTertiary":"#595959",
            "neutralSecondary":"#373737",
            "neutralPrimaryAlt":"#2f2f2f",
            "neutralPrimary":"#000000",
            "neutralDark":"#151515",
            "black":"#0b0b0b",
            "white":"#ffffff",
            "primaryBackground":"#ffffff",
            "primaryText":"#000000",
            "bodyBackground":"#ffffff",
            "bodyText":"#000000",
            "disabledBackground":"#f4f4f4",
            "disabledText":"#c8c8c8"
           }
         }
        },
        {
          "verb": "createSiteColumnXml",
          "schemaXml": "<Field ID=\"{e1605fb4-611a-4eae-b119-77b4b633b436}\" Type=\"MultiChoice\" DisplayName=\"Hobbies\" Required=\"FALSE\" FillInChoice=\"FALSE\" StaticName=\"Hobbies\" Name=\"Hobbies\"><DefaultFormula>=\"\"</DefaultFormula><CHOICES><CHOICE>PingPong</CHOICE><CHOICE>Books</CHOICE><CHOICE>Sharing Is Carring</CHOICE><CHOICE>This is easter egg</CHOICE><CHOICE>Wearing Mask</CHOICE><CHOICE>Face Tatoos</CHOICE><CHOICE>Flying with no Cape</CHOICE></CHOICES></Field>"
        },
        {
          "verb": "createSiteColumn",
          "fieldType": "Text",
          "internalName": "SimpleTextField",
          "displayName": "SimpleTextField",
          "isRequired": false,
          "id": "60f37466-fd00-47a3-9665-44b48a743e27"
        },
        {
          "verb": "createSiteColumn",
          "fieldType": "User",
          "internalName": "siteColumn4User",
          "displayName": "Owner",
          "isRequired": false,
          "id": "181c4370-cdae-471b-9499-730046e55b75"
        },
        {
          "verb": "createSiteColumn",
          "fieldType": "Number",
          "internalName": "Number",
          "displayName": "Number",
          "isRequired": false,
          "id": "151c4370-cdae-471b-9499-730046e55b78"
        },
        {
          "verb": "createContentType",
          "name": "Powershel Samples",
          "description": "Create something with fields",
          "id":"'+$ctId+'",
          "hidden": false,
          "group": "Power Samples",
          "subactions":
            [
              {
                "verb": "addSiteColumn",
                "internalName": "Hobbies"
              },
              {
                "verb": "addSiteColumn",
                "internalName": "SimpleTextField"
              },
              {
                "verb": "addSiteColumn",
                "internalName": "siteColumn4User"
              },
              {
                "verb": "addSiteColumn",
                "internalName": "Number"
              }
            ]
        }
 ],
    "bindata": { },
    "version": "1"
}
```

# [JSON Site Script 2](#tab/json2)
```
{
    "$schema": "https://developer.microsoft.com/json-schemas/sp/site-design-script-actions.schema.json",
    "actions": [
    {
                  "verb": "createSPList",
                  "listName": "DemoListForPowerShellSamples",
                  "templateType": 100,
                  "subactions": [
                  {
                      "verb": "addContentType",
                      "name": "Powershel Samples"
                    },
                    {
                        "verb": "addSPView",
                        "name": "Custom View",
                        "viewFields":
                        [
                          "Hobbies",
                          "Number"
                        ],
                        "query": "<OrderBy><FieldRef Title=\"FileLeafRef\" Ascending=\"TRUE\" /></OrderBy>",
                        "rowLimit": 100,
                        "isPaged": true,
                        "makeDefault": true
                    },
                    {
                        "verb": "removeContentType",
                        "name": "Item"
                    }
                  
                  ]
                  
    }
            
],
     "bindata": { },
    "version": "1"
}
```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

## Contributors

| Author(s) |
|-----------|
| Valeras Narbutas |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-add-site-design-with-custom-list" aria-hidden="true" />

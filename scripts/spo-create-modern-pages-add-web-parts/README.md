

# Create Modern SharePoint Pages and add web parts

## Summary

This sample demonstrates how to create Modern SharePoint Pages without the use of a provisioning engine then add the web parts of the page. 
This shows:

- Creating a page
- Setting up the page header, topic, image and author.
- Adding sections
- Adding Web Parts with some example content


> [!div class="full-image-size"]
> ![Example Screenshot](assets/example.png)

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
[CmdletBinding()]
param (
    [Parameter(Mandatory = $true, HelpMessage = "URL of the SharePoint site")]
    [ValidatePattern('^https://')]
    [string]$SiteUrl,

    [Parameter(Mandatory = $false, HelpMessage = "Name of the page to create")]
    [string]$PageName = "Script-Built-Page.aspx",

    [Parameter(Mandatory = $false, HelpMessage = "Title of the page")]
    [string]$PageTitle = "Baking up a page",

    [Parameter(Mandatory = $false, HelpMessage = "Remove existing page before creating new one")]
    [switch]$CleanExistingPage
)

begin {
    $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
    $logPath = ".\CreateModernPage-$timestamp.log"
    Start-Transcript -Path $logPath

    Write-Host "Starting modern page creation workflow..." -ForegroundColor Cyan

    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate with CLI for Microsoft 365"
    }

    $script:Summary = @{
        SectionsAdded = 0
        WebPartsAdded = 0
        Failures = 0
    }
}

process {
    try {
        if ($CleanExistingPage) {
            Write-Host "Checking for existing page '$PageName'..." -ForegroundColor Yellow
            $removeResult = m365 spo page remove --webUrl $SiteUrl --name $PageName --force 2>&1
            if ($LASTEXITCODE -eq 0) {
                Write-Host "  Removed existing page" -ForegroundColor Green
            }
        }

        Write-Host "Creating page '$PageName'..." -ForegroundColor Cyan
        m365 spo page add --webUrl $SiteUrl --name $PageName --title $PageTitle --layoutType Article
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to create page"
        }
        Write-Host "  Page created successfully" -ForegroundColor Green

        Write-Host "Configuring page header with image and topic..." -ForegroundColor Cyan
        m365 spo page header set --webUrl $SiteUrl --pageName $PageName --type Custom --layout ColorBlock --imageUrl "https://cdn.hubblecontent.osi.office.net/m365content/publish/3c506e10-e846-4698-a041-f133fc505b7b/1140201187.jpg" --topicHeader "Example" --translateX 49.6248124062031 --translateY 37.5 --showTopicHeader
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to set page header"
            $script:Summary.Failures++
        } else {
            Write-Host "  Header configured" -ForegroundColor Green
        }

        Write-Host "Adding section 1 (Two Column Left layout)..." -ForegroundColor Cyan
        m365 spo page section add --webUrl $SiteUrl --pageName $PageName --sectionTemplate TwoColumnLeft --order 1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add section 1"
            $script:Summary.Failures++
        } else {
            $script:Summary.SectionsAdded++
            Write-Host "  Section 1 added" -ForegroundColor Green
        }

        $textContent = "<h2>Welcome to Modern Pages</h2><p>This page demonstrates how to create modern SharePoint pages using CLI for Microsoft 365. You can add rich text content, images, and various web parts to create engaging pages for your users.</p><p>Modern pages provide a responsive, mobile-friendly experience that works seamlessly across devices.</p>"
        
        Write-Host "Adding text content to Section 1, Column 1..." -ForegroundColor Cyan
        m365 spo page text add --webUrl $SiteUrl --pageName $PageName --section 1 --column 1 --order 1 --text $textContent
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add text content"
            $script:Summary.Failures++
        } else {
            $script:Summary.WebPartsAdded++
            Write-Host "  Text content added" -ForegroundColor Green
        }

        Write-Host "Adding Image web part to Section 1, Column 2..." -ForegroundColor Cyan
        m365 spo page clientsidewebpart add --webUrl $SiteUrl --pageName $PageName --standardWebPart Image --section 1 --column 2 --order 1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add Image web part"
            $script:Summary.Failures++
        } else {
            $script:Summary.WebPartsAdded++
            Write-Host "  Image web part added" -ForegroundColor Green
        }

        Write-Host "Adding section 2 (One Column Full Width)..." -ForegroundColor Cyan
        m365 spo page section add --webUrl $SiteUrl --pageName $PageName --sectionTemplate OneColumnFullWidth --order 2
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add section 2"
            $script:Summary.Failures++
        } else {
            $script:Summary.SectionsAdded++
            Write-Host "  Section 2 added" -ForegroundColor Green
        }

        Write-Host "Adding Hero web part to Section 2..." -ForegroundColor Cyan
        $heroProperties = '{"heroLayoutOption":3}'
        m365 spo page clientsidewebpart add --webUrl $SiteUrl --pageName $PageName --standardWebPart Hero --section 2 --column 1 --order 1 --webPartProperties $heroProperties
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add Hero web part"
            $script:Summary.Failures++
        } else {
            $script:Summary.WebPartsAdded++
            Write-Host "  Hero web part added" -ForegroundColor Green
        }

        Write-Host "Adding section 3 (One Column)..." -ForegroundColor Cyan
        m365 spo page section add --webUrl $SiteUrl --pageName $PageName --sectionTemplate OneColumn --order 3
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add section 3"
            $script:Summary.Failures++
        } else {
            $script:Summary.SectionsAdded++
            Write-Host "  Section 3 added" -ForegroundColor Green
        }

        Write-Host "Adding Quick Links web part to Section 3..." -ForegroundColor Cyan
        m365 spo page clientsidewebpart add --webUrl $SiteUrl --pageName $PageName --standardWebPart QuickLinks --section 3 --column 1 --order 1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to add Quick Links web part"
            $script:Summary.Failures++
        } else {
            $script:Summary.WebPartsAdded++
            Write-Host "  Quick Links web part added" -ForegroundColor Green
        }

        Write-Host "Publishing page..." -ForegroundColor Cyan
        m365 spo page publish --webUrl $SiteUrl --name $PageName
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to publish page"
            $script:Summary.Failures++
        } else {
            Write-Host "  Page published successfully" -ForegroundColor Green
        }

    }
    catch {
        Write-Error "Error during page creation: $_"
        $script:Summary.Failures++
    }
}

end {
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Page Creation Summary" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    Write-Host "Page Name       : $PageName"
    Write-Host "Sections Added  : $($script:Summary.SectionsAdded)"
    Write-Host "Web Parts Added : $($script:Summary.WebPartsAdded)"
    
    if ($script:Summary.Failures -gt 0) {
        Write-Host "Failures        : $($script:Summary.Failures)" -ForegroundColor Red
    } else {
        Write-Host "Failures        : $($script:Summary.Failures)" -ForegroundColor Green
    }
    
    Write-Host "Page URL        : $SiteUrl/SitePages/$PageName" -ForegroundColor Cyan
    Write-Host "========================================`n" -ForegroundColor Cyan

    Stop-Transcript
}

# Basic usage
# .\Create-ModernPage.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing"

# Create page with custom name and title
# .\Create-ModernPage.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -PageName "TeamPage.aspx" -PageTitle "Team Collaboration Hub"

# Remove existing page and create new one
# .\Create-ModernPage.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -CleanExistingPage

# Create page with verbose output
# .\Create-ModernPage.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/Marketing" -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

# [PnP PowerShell](#tab/pnpps)

```powershell

[CmdletBinding()]
param (
  [string]$Tenant = "contoso",
  [string]$Site = "Intranet",
  [switch]$CleanExistingPage
)
begin{

    Connect-PnPOnline "https://$($Tenant).sharepoint.com/sites/$($Site)" -Interactive

    # Add Page
    $pageParams = @{
        Name = "Script-Built-Page.aspx"
        Title = "Baking up a page"
        HeaderLayoutType = "ColorBlock"
    }

}
process {

    if($CleanExistingPage){
        # Clean Up if repeating script
        Write-Host "Removing existing page..." -ForegroundColor Yellow
        Remove-PnPPage $pageParams.Name -Force
    }
        
    Write-Host "Adding new page..."
    $newPage = Add-PnPPage @pageParams

    # Set the header, topic, image, and person
    Write-Host "Setting the header, topic, image, and person..."
    
    # You can use ?maintenancemode=true on an existing page to help get these values or Use Get-PnPPage and look through the properties in the controls object.
    $newPage.PageHeader.ImageServerRelativeUrl = "https://cdn.hubblecontent.osi.office.net/m365content/publish/3c506e10-e846-4698-a041-f133fc505b7b/1140201187.jpg"
    $newPage.PageHeader.TopicHeader = "Example"
    $newPage.PageHeader.Show
    $newPage.PageHeader.TranslateX = 49.6248124062031
    $newPage.PageHeader.TranslateY = 37.5
    
    # Authors details will reference the target tenant e.g. domain etc.
    $newPage.PageHeader.Authors = '[{"id":"AdeleV@contoso.co.uk","email":"AdeleV@contoso.co.uk","name":"Adele Vance","role":"Retail Manager"}]'
    $newPage.PageHeader.AuthorByLine = '["AdeleV@contoso.co.uk"]'

    # Save the page
    $newPage.Save()

    # Refresh variable
    $newPage = Get-PnPPage $pageParams.Name

    # Create a one-third right section
    # Add Text, Image
    Write-Host "Row 1 - Setting the section and web parts"
    $newPage | Add-PnPPageSection -SectionTemplate TwoColumnLeft -Order 1

    # Column 1 - Text Web Part
    $newPage | Add-PnPPageTextPart -Order 1 -Column 1 -Section 1 `
                -Text @"
                        <h2>Welcome to the Page Bakery 😊</h2>
                        <p>Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Maecenas porttitor congue massa. Fusce posuere, magna sed pulvinar ultricies, purus lectus malesuada libero, sit amet commodo magna eros quis urna. </p>
                        <p><em>Nunc viverra imperdiet enim. Fusce est. Vivamus a tellus. Pellentesque habitant morbi tristique senectus et netus et malesuada fames ac turpis egestas. Proin pharetra nonummy pede. Mauris et orci. Aenean nec lorem. In porttitor. Donec laoreet nonummy augue.​​​​​​​</em></p>
"@

    # Column 2 - Image Web Part

    # The referenced image uses a stock image
    $propsBreadImg = '{"title": "Image", "description": "Image", "dataVersion": "1.8", "properties": {"imageSourceType":2,"altText":"","overlayText":"","imgWidth":5616,"imgHeight":3744,"fixAspectRatio":false}, "serverProcessedContent": {"searchablePlainTexts":{"captionText":"Baking a lovely page"},"imageSources":{"imageSource":"https://cdn.hubblecontent.osi.office.net/m365content/publish/005eb6ca-fe86-4433-921c-126cb23c7adb/576678384.jpg"},"links":{}}}'

    $newPage | Add-PnPPageWebPart -Order 1 -Column 2 -Section 1 -DefaultWebPartType Image -WebPartProperties $propsBreadImg

    

    # Create a one-third left section
    # Add Text, Image
    Write-Host "Row 2 - Setting the section and web parts"
    $newPage | Add-PnPPageSection -SectionTemplate TwoColumnRight -Order 2 -ZoneEmphasis 2

    # Column 1 - Image Web Part
    #  Use Get-PnPPage and look through the properties in the controls object, if you want a reference
    $propsBookImg = '{"title": "Image", "description": "Image", "dataVersion": "1.8", "properties": {"imageSourceType":2,"altText":"","overlayText":"","imgWidth":5616,"imgHeight":3744,"fixAspectRatio":false}, "serverProcessedContent": {"searchablePlainTexts":{"captionText":"Showing pages some love"},"imageSources":{"imageSource":"https://cdn.hubblecontent.osi.office.net/m365content/publish/0d3ff8a6-8854-4a6a-aee1-b1c7d04bb857/691081033.jpg"},"links":{}}}'

    $newPage | Add-PnPPageWebPart -Order 1 -Column 1 -Section 2 -DefaultWebPartType Image -WebPartProperties $propsBookImg

    # Column 2 - Text Web Part
    $newPage | Add-PnPPageTextPart -Order 1 -Column 2 -Section 2 `
    -Text @"
            <h2>How you can improve your pages?</h2>
            <p>Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Maecenas porttitor congue massa. Fusce posuere, magna sed pulvinar ultricies, purus lectus malesuada libero, sit amet commodo magna eros quis urna. </p>
            <p><strong><em>Nunc viverra imperdiet enim. Fusce est. Vivamus a tellus. Pellentesque habitant morbi tristique senectus et netus et malesuada fames ac turpis egestas. Proin pharetra nonummy pede. Mauris et orci. Aenean nec lorem. In porttitor. Donec laoreet nonummy augue.​​​​​​​<strong></em></p>
"@
    

    # Create single column section
    # Add Quick Links inc Icons

    Write-Host "Row 3 - Setting the section and web parts"
    $newPage | Add-PnPPageSection -SectionTemplate OneColumn -Order 3 -ZoneEmphasis 3

    # Column 1 - Quicklinks Web Part
    # You can use ?maintenancemode=true on an existing page to help get these values or Use Get-PnPPage and look through the properties in the controls object.
    $propsLinksWebPart = @"
    {
        "title": "Quick links",
        "description": "Show a collection of links to content such as documents, images, videos, and more in a variety of layouts with options for icons, images, and audience targeting.",
        "serverProcessedContent": {
          "searchablePlainTexts": {
            "title": "Interesting resources",
            "items[0].title": "How to make a page",
            "items[1].title": "Bringing pages into modern",
            "items[2].title": "Accessibility Considerations in Pages",
            "items[3].title": "Showing your page some love with images"
          },
          "links": {
            "baseUrl": "/sites/Intranet",
            "items[0].sourceItem.url": "/sites/Intranet/SitePages/Homepage-Example.aspx",
            "items[1].sourceItem.url": "/sites/Intranet/SitePages/Homepage-Example.aspx",
            "items[2].sourceItem.url": "/sites/Intranet/SitePages/Homepage-Example.aspx",
            "items[3].sourceItem.url": "/sites/Intranet/SitePages/Homepage-Example.aspx"
          }
        },
        "dataVersion": "2.2",
        "properties": {
          "items": [
            {
              "thumbnailType": 2,
              "id": 4,
              "description": "",
              "fabricReactIcon": { "iconName": "glimmer"},
              "altText": "",
              "rawPreviewImageMinCanvasWidth": 32767
            },
            {
              "thumbnailType": 2,
              "id": 3,
              "description": "",
              "fabricReactIcon": { "iconName": "webappbuilderfragment" },
              "altText": "",
              "rawPreviewImageMinCanvasWidth": 32767
            },
            {
              "thumbnailType": 2,
              "id": 2,
              "description": "",
              "fabricReactIcon": { "iconName": "group" },
              "altText": "",
              "rawPreviewImageMinCanvasWidth": 32767
            },
            {
              "thumbnailType": 2,
              "id": 1,
              "description": "",
              "fabricReactIcon": { "iconName": "heartfill" },
              "altText": "",
              "rawPreviewImageMinCanvasWidth": 32767
            }
          ],
          "isMigrated": true,
          "layoutId": "CompactCard",
          "shouldShowThumbnail": true,
          "imageWidth": 100,
          "buttonLayoutOptions": {
            "showDescription": false,
            "buttonTreatment": 2,
            "iconPositionType": 2,
            "textAlignmentVertical": 2,
            "textAlignmentHorizontal": 2,
            "linesOfText": 2
          },
          "listLayoutOptions": { "showDescription": false, "showIcon": true },
          "waffleLayoutOptions": { "iconSize": 1, "onlyShowThumbnail": false },
          "hideWebPartWhenEmpty": true,
          "dataProviderId": "QuickLinks",
          "iconPicker": "glimmer"
        }
      }
"@

    $newPage | Add-PnPPageWebPart -Order 1 -Column 1 -Section 3 -DefaultWebPartType QuickLinks -WebPartProperties $propsLinksWebPart

    Write-Host "Publishing Page and promote Page as News..."
    $newPage | Set-PnPPage -PromoteAs NewsArticle -Publish

}
end{

  Write-Host "Done! :)" -ForegroundColor Green
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
***


## Contributors

| Author(s) |
|-----------|
| Paul Bullock |
| [Adam Wójcik](https://github.com/Adam-it) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-create-modern-pages-add-web-parts" aria-hidden="true" />

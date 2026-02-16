

# Translate columns in a SharePoint list

## Summary

Say we have PowerApp that is storing its data in SharePoint lists. We want to make the app multilingual. This sample shows the process using PnP PowerShell or CLI for Microsoft 365 v11.4.0+. We Add a new list with the languages we want to support called ID_Languages:
![Example Screenshot](assets/Languages.PNG)

We then add a Radio control to our list to let the user select a language. We set the onChange of the control to :
Set(gvLanguage,Radio1.Selected.Title);

So now we have the selected language in gvLanguage.

Next we want to change all the labels in the app to display the translated text of the label. So we create a table called ID-App Labels

![Example Screenshot](assets/AppLabels.PNG)

The Title field is what we use in code to find the correct label (If you're using forms make it the same as the name of the field). The Language is a lookup column pointing back to our ID-Languages list. The Translation column is the translated version of that label. We just add the values for our native language for now.

We add the following to the OnSelect of the Language Radio control to get all the labels for the selected language:
Set(
    gvLabels,
    Filter(
        'ID-App Labels',
        Language.Value = gvLanguage
    )
);


Now, whenever we want to add literal text to our PowerApp , instead of typing in the text we use the formula
First(
    Filter(
        gvLabels,
        Title = "AppTitle"
    )
).Translation

Next for any dropdown lists we create a new list to hold the translated values. For example here is a list (ID-Ease) used in dropdowns:

![Example Screenshot](assets/EaSE.PNG)

Again we only add the values for our native language now. Again the Language column is a lookup to our Languages list. The MasterEase column is a lookup to a master list of Ease values. For a single lookup value, all the translations must point back to the same master record for reporting purposes(That's not important for this sample though).

Now , when we want to display the "Ease" is a Dropdown or ComboBox we set the items to 

Filter('ID-Ease',Language.Value=gvLanguage)

Now for the scripting part. We get everything developed and working in our native language and we want to add the
translations for all the other languages in or ID-Languages list.


# [PnP PowerShell](#tab/pnpps)

```powershell
Connect-PnPOnline "https://tenant.sharepoint.com/sites/ShopFloorIdeation/" -DeviceLogin
function Translate-List {
    param (
        [string]$ListId, ## the ID or Title of the list to translate
        [string[]]$ColumnsToTranslate, ## the names of the columns to translate
        [string[]]$ColumnsToCopy, ## the names of the columns to copy without translating
        [string]$LanguageColumnName, ## the name of the Language column (must be a lookup)
        [string]$FromLanguage, ## the language we are translating from (items in the list must be of this lanugage)
        [string]$LanguageList ## the ID or Title of the List of languages
    )
    $endpoint = "https://api.cognitive.microsofttranslator.com/"
    $subscriptionkey = "YOURKEY"
    $headers = @{"Ocp-Apim-Subscription-Key" = $subscriptionkey; "Content-Type" = "application/json"; } 
    ##1. Sanity check
    $list = get-pnpList -Identity $ListId
    $fields = Get-PNPField -List $list
    if ($null -eq $list) {
        Write-Error "List $listId not found"
        return;
    }
    foreach ($col in $ColumnsToCopy) {
        $field = $Fields.Where({ $_.InternalName -eq $col }) 
        if (0 -eq $field.Count) {
            Write-Error "Field $col not found in list $listId"
            return;
        }
        $fieldType = $field.FieldTypeKind
        if ($fieldType -ne "Lookup" -and $fieldType -ne "Text") {
            Write-Error "Cannot copy field $col of type  $fieldType"
            return;
        }
    }
    foreach ($col in $ColumnsToTranslate) {
        $field = $Fields.Where({ $_.InternalName -eq $col }) 
        if (0 -eq $field.Count) {
            Write-Error "Field $col not found in list $listId"
            return;
        }
        $fieldType = $field.FieldTypeKind
        if ($fieldType -ne "Text") {
            Write-Error "Cannot translate field $col of type  $fieldType"
            return;
        }
    }

    ##2. Remove old items
    
    $items = (Get-PnPListItem -List $list -PageSize 10000)
    $deletebatch = new-PnPBatch
    foreach ($item in $items) {
        If ($item.FieldValues[$LanguageColumnName].LookupValue -ne $FromLanguage) {
            Remove-PnPListItem -List $list -Identity $item.Id -Batch $deletebatch -Recycle
        }
        if ($deletebatch.RequestCount -eq 100) {
            Invoke-PnpBatch $deletebatch
            $deletebatch = new-PnPBatch
        }
    }
    if ($deletebatch.RequestCount -gt 0) {
        Invoke-PnpBatch $deletebatch
    }
    ##3. Add back translations

    $items = (Get-PnPListItem -List $list -PageSize 10000)
    $languages = (Get-PnPListItem -List (get-pnpList -Identity "ID-Languages") -PageSize 10000)
    
    foreach ($language in $languages | Where-Object { $_.FieldValues["Title"] -ne $FromLanguage }) {
        $route = "/translate?api-version=3.0&from=$FromLanguage&to=" + $language.FieldValues["Title"]
        $uri = "$endpoint$route"
        foreach ($item in $items) {
            $updates = @{
                $LanguageColumnName = $language.FieldValues["ID"]
            }
            foreach ($col in $ColumnsToCopy) {
                $field = $Fields.Where({ $_.InternalName -eq $col }) 
                $fieldType = $field.FieldTypeKind
                switch ($fieldType) {
                    "Lookup" { 
                        $updates[$col] = $item[$col].LookupId
                    }
                    "Text" { 
                        $updates[$col] = $item[$col]
                    }
                    Default {
                        Write-Host "Cannot copy field $col of type  $fieldType"
                    }
                }
            }
            foreach ($ColumnToTranslate in $ColumnsToTranslate) {
                if ($null -eq $item.FieldValues[$ColumnToTranslate] ) {
                    $updates[$ColumnToTranslate] = $null
                }
                else {
                    $body = @(
                        @{
                            "Text" = $item.FieldValues[$ColumnToTranslate]
                        })
                    $body = ConvertTo-Json $body
                    $response = Invoke-WebRequest -Uri $uri -Body $body  -Headers $headers -Method Post
                    $xlat = $response | ConvertFrom-Json
                    if ($xlat.translations.text.Length -lt 256) {
                        $translation = $xlat.translations.text
                    }
                    else {
                        $translation = $xlat.translations.text.Substring(0, 255)
                    }
                    $updates[$ColumnToTranslate] = $translation
                }    
            }
            $newItem = Add-PnPListItem -List $list  -Values $updates
        }
    }
    return 
}
Translate-List -ListId "ID-App Labels" -ColumnsToTranslate @("Translation")  -ColumnsToCopy @("Title") -LanguageColumnName "Language" -FromLanguage "en" -LanguageList "ID-Languages"
Translate-List -ListId "ID-Ease" -ColumnsToTranslate @("Title")  -ColumnsToCopy @("MasterEase") -LanguageColumnName "Language" -FromLanguage "en" -LanguageList "ID-Languages"

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365)

```powershell
[CmdletBinding()]
param(
  [Parameter(Mandatory, HelpMessage="SharePoint site URL where the lists reside")]
  [string]$SiteUrl,

  [Parameter(Mandatory, HelpMessage="ID or title of the list to translate")]
  [string]$ListId,

  [Parameter(Mandatory, HelpMessage="Array of column names to translate")]
  [string[]]$ColumnsToTranslate,

  [Parameter(Mandatory, HelpMessage="Array of column names to copy without translating")]
  [string[]]$ColumnsToCopy,

  [Parameter(Mandatory, HelpMessage="Name of the Language column (must be a Lookup field)")]
  [string]$LanguageColumnName,

  [Parameter(Mandatory, HelpMessage="Source language code (e.g., 'en', 'de', 'fr')")]
  [string]$FromLanguage,

  [Parameter(Mandatory, HelpMessage="ID or title of the Languages list")]
  [string]$LanguageList,

  [Parameter(Mandatory, HelpMessage="Azure Translator subscription key")]
  [string]$TranslatorKey,

  [Parameter(HelpMessage="Path where transcript and report will be saved")]
  [string]$OutputPath = (Get-Location).Path
)

begin {
  $transcriptPath = Join-Path $OutputPath "TranslateList_$(Get-Date -Format 'yyyyMMdd_HHmmss').log"
  Start-Transcript -Path $transcriptPath

  Write-Host "[1/6] Authenticating with Microsoft 365..." -ForegroundColor Cyan
  m365 login --ensure
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to authenticate with Microsoft 365"
  }

  $script:Summary = @{
    ItemsRemoved = 0
    ItemsCreated = 0
    Failures = 0
  }

  Write-Host "[2/6] Validating list and fields..." -ForegroundColor Cyan

  $listJson = m365 spo list get --webUrl $SiteUrl --title $ListId --output json
  if ($LASTEXITCODE -ne 0) {
    throw "List '$ListId' not found at $SiteUrl"
  }
  $list = $listJson | ConvertFrom-Json

  $fieldsJson = m365 spo field list --webUrl $SiteUrl --listTitle $ListId --output json
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to retrieve fields for list '$ListId'"
  }
  $fields = @($fieldsJson | ConvertFrom-Json)

  foreach ($col in $ColumnsToCopy) {
    $field = $fields | Where-Object { $_.InternalName -eq $col }
    if (-not $field) {
      throw "Field '$col' not found in list '$ListId'"
    }
    if ($field.FieldTypeKind -ne 2 -and $field.FieldTypeKind -ne 7) {
      throw "Cannot copy field '$col' of type $($field.TypeAsString). Only Text (2) and Lookup (7) are supported."
    }
  }

  foreach ($col in $ColumnsToTranslate) {
    $field = $fields | Where-Object { $_.InternalName -eq $col }
    if (-not $field) {
      throw "Field '$col' not found in list '$ListId'"
    }
    if ($field.FieldTypeKind -ne 2) {
      throw "Cannot translate field '$col' of type $($field.TypeAsString). Only Text (2) is supported."
    }
  }

  Write-Host "  Validation complete: All fields exist and have compatible types" -ForegroundColor Green
}

process {
  Write-Host "[3/6] Retrieving existing items from list..." -ForegroundColor Cyan
  $itemsJson = m365 spo listitem list --webUrl $SiteUrl --listTitle $ListId --output json
  if ($LASTEXITCODE -ne 0) {
    Write-Warning "Failed to retrieve items from list '$ListId'"
    $script:Summary.Failures++
    return
  }
  $items = @($itemsJson | ConvertFrom-Json)
  Write-Host "  Found $($items.Count) existing items" -ForegroundColor Gray

  Write-Host "[4/6] Removing old translated items (preserving source language)..." -ForegroundColor Cyan
  $itemsToRemove = @()
  $languageField = $fields | Where-Object { $_.InternalName -eq $LanguageColumnName }

  foreach ($item in $items) {
    $languageValue = $null
    if ($languageField.FieldTypeKind -eq 7) {
      $languageValue = $item."$LanguageColumnName`LookupValue"
    }
    else {
      $languageValue = $item.$LanguageColumnName
    }

    if ($languageValue -ne $FromLanguage) {
      $itemsToRemove += $item.ID
    }
  }

  if ($itemsToRemove.Count -gt 0) {
    Write-Verbose "  Removing $($itemsToRemove.Count) translated items..."
    for ($i = 0; $i -lt $itemsToRemove.Count; $i += 100) {
      $batch = $itemsToRemove[$i..[Math]::Min($i + 99, $itemsToRemove.Count - 1)]
      $idsParam = ($batch -join ',')
      m365 spo listitem batch remove --webUrl $SiteUrl --listTitle $ListId --ids $idsParam --recycle
      if ($LASTEXITCODE -eq 0) {
        $script:Summary.ItemsRemoved += $batch.Count
        Write-Verbose "    Removed batch of $($batch.Count) items"
      }
      else {
        Write-Warning "    Failed to remove batch of items"
        $script:Summary.Failures++
      }
    }
  }
  else {
    Write-Host "  No translated items to remove" -ForegroundColor Gray
  }

  Write-Host "[5/6] Retrieving languages and source items..." -ForegroundColor Cyan
  $languagesJson = m365 spo listitem list --webUrl $SiteUrl --listTitle $LanguageList --output json
  if ($LASTEXITCODE -ne 0) {
    Write-Warning "Failed to retrieve languages from list '$LanguageList'"
    $script:Summary.Failures++
    return
  }
  $languages = @($languagesJson | ConvertFrom-Json) | Where-Object { $_.Title -ne $FromLanguage }

  $sourceItemsJson = m365 spo listitem list --webUrl $SiteUrl --listTitle $ListId --output json
  if ($LASTEXITCODE -ne 0) {
    Write-Warning "Failed to retrieve source items from list '$ListId'"
    $script:Summary.Failures++
    return
  }
  $sourceItems = @($sourceItemsJson | ConvertFrom-Json)

  Write-Host "[6/6] Translating and creating items for $($languages.Count) languages..." -ForegroundColor Cyan
  $totalOperations = $languages.Count * $sourceItems.Count
  $currentOperation = 0

  foreach ($language in $languages) {
    Write-Host "  Translating to: $($language.Title)" -ForegroundColor Yellow
    $targetLanguage = $language.Title

    $csvRows = @()

    foreach ($item in $sourceItems) {
      $currentOperation++
      Write-Progress -Activity "Translating list items" -Status "Language: $targetLanguage | Item $currentOperation of $totalOperations" -PercentComplete (($currentOperation / $totalOperations) * 100)

      try {
        $csvRow = @{}
        $csvRow[$LanguageColumnName] = $language.ID

        foreach ($col in $ColumnsToCopy) {
          $field = $fields | Where-Object { $_.InternalName -eq $col }
          switch ($field.FieldTypeKind) {
            7 {
              $lookupField = $item."$col`LookupId"
              if ($lookupField) {
                $csvRow[$col] = $lookupField
              }
            }
            2 {
              if ($item.$col) {
                $csvRow[$col] = $item.$col
              }
            }
          }
        }

        foreach ($colToTranslate in $ColumnsToTranslate) {
          if (-not $item.$colToTranslate) {
            $csvRow[$colToTranslate] = $null
          }
          else {
            $translationUrl = "https://api.cognitive.microsofttranslator.com/translate?api-version=3.0&from=$FromLanguage&to=$targetLanguage"
            $headers = @{
              "Ocp-Apim-Subscription-Key" = $TranslatorKey
              "Content-Type"               = "application/json"
            }
            $headersJson = $headers | ConvertTo-Json -Compress
            $body = @(@{ "Text" = $item.$colToTranslate }) | ConvertTo-Json -Compress

            $translationJson = m365 request --method POST --url $translationUrl --headers $headersJson --body $body --output json
            if ($LASTEXITCODE -eq 0) {
              $translationResponse = $translationJson | ConvertFrom-Json
              $translation = $translationResponse[0].translations[0].text
              if ($translation.Length -gt 255) {
                $translation = $translation.Substring(0, 255)
              }
              $csvRow[$colToTranslate] = $translation
            }
            else {
              Write-Warning "    Failed to translate field '$colToTranslate' for item ID $($item.ID)"
              $csvRow[$colToTranslate] = $item.$colToTranslate
            }
          }
        }

        $csvRows += $csvRow
      }
      catch {
        Write-Warning "    Error processing item ID $($item.ID): $_"
        $script:Summary.Failures++
        continue
      }
    }

    if ($csvRows.Count -gt 0) {
      Write-Verbose "    Creating $($csvRows.Count) translated items via batch..."
      $csvContent = $csvRows | ForEach-Object {
        $obj = [PSCustomObject]$_
        $obj
      } | ConvertTo-Csv -NoTypeInformation | Out-String

      $tempCsvPath = Join-Path $env:TEMP "translate_batch_$(Get-Date -Format 'yyyyMMddHHmmss').csv"
      $csvContent | Out-File -FilePath $tempCsvPath -Encoding UTF8

      m365 spo listitem batch add --webUrl $SiteUrl --listTitle $ListId --filePath $tempCsvPath
      if ($LASTEXITCODE -eq 0) {
        $script:Summary.ItemsCreated += $csvRows.Count
        Write-Host "    Created $($csvRows.Count) items for language: $targetLanguage" -ForegroundColor Green
      }
      else {
        Write-Warning "    Failed to create batch of items for language: $targetLanguage"
        $script:Summary.Failures++
      }

      Remove-Item -Path $tempCsvPath -Force -ErrorAction SilentlyContinue
    }
  }

  Write-Progress -Activity "Translating list items" -Completed
}

end {
  Stop-Transcript

  Write-Host "`n===============================================" -ForegroundColor Cyan
  Write-Host "           TRANSLATION SUMMARY" -ForegroundColor Cyan
  Write-Host "===============================================" -ForegroundColor Cyan
  Write-Host "Items Removed:  $($script:Summary.ItemsRemoved)" -ForegroundColor $(if ($script:Summary.ItemsRemoved -gt 0) { 'Yellow' } else { 'Gray' })
  Write-Host "Items Created:  $($script:Summary.ItemsCreated)" -ForegroundColor $(if ($script:Summary.ItemsCreated -gt 0) { 'Green' } else { 'Gray' })
  Write-Host "Failures:       $($script:Summary.Failures)" -ForegroundColor $(if ($script:Summary.Failures -gt 0) { 'Red' } else { 'Gray' })
  Write-Host "===============================================" -ForegroundColor Cyan
  Write-Host "Transcript saved to: $transcriptPath" -ForegroundColor Gray
}

# Example 1: Basic translation of app labels
# ./Translate-List.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/MyPowerApp" -ListId "ID-App Labels" -ColumnsToTranslate @("Translation") -ColumnsToCopy @("Title") -LanguageColumnName "Language" -FromLanguage "en" -LanguageList "ID-Languages" -TranslatorKey "YOUR_AZURE_KEY"

# Example 2: Translate with verbose output
# ./Translate-List.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/MyPowerApp" -ListId "ID-Ease" -ColumnsToTranslate @("Title") -ColumnsToCopy @("MasterEase") -LanguageColumnName "Language" -FromLanguage "en" -LanguageList "ID-Languages" -TranslatorKey "YOUR_AZURE_KEY" -Verbose

# Example 3: Translate with custom output path
# ./Translate-List.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/MyPowerApp" -ListId "ID-App Labels" -ColumnsToTranslate @("Translation") -ColumnsToCopy @("Title") -LanguageColumnName "Language" -FromLanguage "en" -LanguageList "ID-Languages" -TranslatorKey "YOUR_AZURE_KEY" -OutputPath "C:\Reports"

# Example 4: Translate multiple columns
# ./Translate-List.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/MyPowerApp" -ListId "ID-Products" -ColumnsToTranslate @("ProductName","Description") -ColumnsToCopy @("SKU","Category") -LanguageColumnName "Language" -FromLanguage "en" -LanguageList "ID-Languages" -TranslatorKey "YOUR_AZURE_KEY"

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

After running the script the translated values have been added to your ID-App Labels and ID-Ease lists:

![Example Screenshot](assets/AppLabelsTranslated.PNG)

![Example Screenshot](assets/EaseTranslated.PNG)

If you add new items to any of the lists the script can be rerun and it will re-translate everything.

To run the script you will need to get a key for Azure translation services as described at https://learn.microsoft.com/azure/ai-services/translator/create-translator-resource. Your key should be place in the variable $subscriptionkey 

> [!Note]
> The translations done using machine translation services should be reviewed by someone who speaks both languages. The are sometime incorrect.


***

## Contributors

| Author(s) |
|-----------|
| Russell Gove |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-translate-list" aria-hidden="true" />

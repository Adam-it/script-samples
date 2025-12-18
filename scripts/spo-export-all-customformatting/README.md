

# Extract all custom formatting

## Summary

This script (available in both PnP PowerShell and CLI for Microsoft 365) stems from a scenario where a client had wiped out some column formatting I had built, and I realized that there was no good way to get it back.

This script will scrape every list, field, view and ContentType form on the site, and save it in a folder structure that's easy to navigate, and to store in your DevOps repo, or other version control, so you can easily restore it if it gets wiped out, or just see what has changed.

# [PnP PowerShell](#tab/pnpps)

```powershell
function get-customFormatting() {
    $url = Read-Host -Prompt "Enter the URL of the site you wish to backup custom formatting from"
    Connect-PnPOnline $url -Interactive

    try {
        $web = Get-PnPWeb -Includes Title
    }
    catch {
        Write-Host "Please connect to a site first" -Color Red
        return;
    }

    Write-Host "Backing up formatting for '$($web.Title)', fetching lists";

    $lists = Get-PnPList -Includes Id, Title, Views, Fields, ContentTypes | Where-Object { -not $_.Hidden }

    Write-Host "Fetched data - starting backup";


    foreach ($list in $lists) {
        $fields = $list.Fields | Where-Object { $_.CustomFormatter -ne $null -and $_.CustomFormatter -ne "" }
    
        foreach ($field in $fields) {
            try {
                Write-Host "List '$($list.Title)' > field: '$($field.Title)'";
                New-Item -Path "CustomFormatting\$($list.Title)\Columns\" -Name "$($field.Title) ($($field.InternalName)).column-formatter.json" -ItemType File -Value $($field.CustomFormatter | ConvertFrom-Json | ConvertTo-Json -Depth 100) -Force | Out-Null;
            }
            catch {
                Write-Host "Error: $($_.Exception.Message)" -ForegroundColor Red;
            }
        }

        $views = $list.Views | Where-Object { $_.CustomFormatter -ne $null -and $_.CustomFormatter -ne "" }
        foreach ($view in $views) {
            try {
                Write-Host "List '$($list.Title)' > `View: '$($view.Title)'";
                New-Item -Path "CustomFormatting\$($list.Title)\Views\" -Name "$($view.Title).view-formatter.json" -ItemType File -Value $($view.CustomFormatter | ConvertFrom-Json | ConvertTo-Json -Depth 100) -Force | Out-Null;
            }
            catch {
                Write-Host "Error: $($_.Exception.Message)" -ForegroundColor Red;
            }
        }

        $formCustomizer = $list.ContentTypes | Where-Object { $_.ClientFormCustomFormatter -ne $null -and $_.ClientFormCustomFormatter -ne "" }
        foreach ($form in $formCustomizer) {
            try {
                Write-Host "List '$($list.Title)' > form: '$($form.Name)'";
                New-Item -Path "CustomFormatting\$($list.Title)\Forms\" -Name "$($form.Name).form-formatter.json" -ItemType File -Value $($form.ClientFormCustomFormatter | ConvertFrom-Json | ConvertTo-Json -Depth 100) -Force | Out-Null;
            }
            catch {
                Write-Host "Error: $($_.Exception.Message)" -ForegroundColor Red;
            }
        }
    }

}

get-customFormatting;
```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
function Export-SPOCustomFormatting {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory = $true, HelpMessage = 'The URL of the SharePoint site from which to export custom formatting')]
        [string]$SiteUrl,

        [Parameter(Mandatory = $false, HelpMessage = 'The local folder path where custom formatting JSON files will be saved. Defaults to "CustomFormatting" in the current directory')]
        [string]$OutputPath = 'CustomFormatting',

        [Parameter(Mandatory = $false, HelpMessage = 'Path to export detailed results to CSV for audit and reporting')]
        [string]$ExportCsvPath
   )

    begin {
        Write-Verbose 'Ensuring user is logged in to CLI for Microsoft 365'
        m365 login --ensure 2>&1 | Out-Null
        if ($LASTEXITCODE -ne 0) {
            throw 'Failed to authenticate to CLI for Microsoft 365. Please check your credentials and try again.'
        }

        $script:Summary = [ordered]@{
            ListsProcessed  = 0
            FieldsExported  = 0
            ViewsExported   = 0
            FormsExported   = 0
            Failures        = @()
        }
    }

    process {
        Write-Host "Fetching lists from '$SiteUrl'"

        $listsJson = m365 spo list list --webUrl $SiteUrl --query "[?Hidden == \`false\`]" --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to retrieve lists. CLI output: $listsJson"
        }

        $lists = @($listsJson | ConvertFrom-Json)
        Write-Host "Found $($lists.Count) non-hidden list(s). Starting backup..."

        foreach ($list in $lists) {
            $script:Summary.ListsProcessed++

            # Export fields with custom formatting
            try {
                Write-Verbose "Processing fields for list '$($list.Title)'"
                $fieldsJson = m365 spo field list --webUrl $SiteUrl --listId $list.Id --output json 2>&1
               if ($LASTEXITCODE -eq 0) {
                    $fields = @($fieldsJson | ConvertFrom-Json) | Where-Object { -not [string]::IsNullOrEmpty($_.CustomFormatter) }

                    foreach ($field in $fields) {
                        $folderPath = Join-Path $OutputPath "$($list.Title)\Columns"
                        $fileName = "$($field.Title) ($($field.InternalName)).column-formatter.json"
                        $filePath = Join-Path $folderPath $fileName

                        if ($PSCmdlet.ShouldProcess($filePath, 'Create custom formatter file')) {
                            try {
                                $null = New-Item -Path $folderPath -ItemType Directory -Force
                                $field.CustomFormatter | ConvertFrom-Json | ConvertTo-Json -Depth 100 | Set-Content -Path $filePath
                                Write-Host "  Exported field: '$($field.Title)'" -ForegroundColor Green
                                $script:Summary.FieldsExported++
                            }
                            catch {
                                Write-Warning "Failed to export field '$($field.Title)': $($_.Exception.Message)"
                                $script:Summary.Failures += [pscustomobject]@{ List = $list.Title; Type = 'Field'; Name = $field.Title; Error = $_.Exception.Message }
                            }
                        }
                    }
                }
            }
            catch {
                Write-Warning "Failed to process fields for list '$($list.Title)': $($_.Exception.Message)"
                $script:Summary.Failures += [pscustomobject]@{ List = $list.Title; Type = 'Fields'; Name = 'N/A'; Error = $_.Exception.Message }
            }

            # Export views with custom formatting
            try {
                Write-Verbose "Processing views for list '$($list.Title)'"
                $viewsJson = m365 spo list view list --webUrl $SiteUrl --listId $list.Id --output json 2>&1
               if ($LASTEXITCODE -eq 0) {
                    $views = @($viewsJson | ConvertFrom-Json) | Where-Object { -not [string]::IsNullOrEmpty($_.CustomFormatter) }

                    foreach ($view in $views) {
                        $folderPath = Join-Path $OutputPath "$($list.Title)\Views"
                        $fileName = "$($view.Title).view-formatter.json"
                        $filePath = Join-Path $folderPath $fileName

                        if ($PSCmdlet.ShouldProcess($filePath, 'Create custom formatter file')) {
                            try {
                                $null = New-Item -Path $folderPath -ItemType Directory -Force
                                $view.CustomFormatter | ConvertFrom-Json | ConvertTo-Json -Depth 100 | Set-Content -Path $filePath
                                Write-Host "  Exported view: '$($view.Title)'" -ForegroundColor Green
                                $script:Summary.ViewsExported++
                            }
                            catch {
                                Write-Warning "Failed to export view '$($view.Title)': $($_.Exception.Message)"
                                $script:Summary.Failures += [pscustomobject]@{ List = $list.Title; Type = 'View'; Name = $view.Title; Error = $_.Exception.Message }
                            }
                        }
                    }
                }
            }
            catch {
                Write-Warning "Failed to process views for list '$($list.Title)': $($_.Exception.Message)"
                $script:Summary.Failures += [pscustomobject]@{ List = $list.Title; Type = 'Views'; Name = 'N/A'; Error = $_.Exception.Message }
            }

            # Export content types with custom form formatting
            try {
                Write-Verbose "Processing content types for list '$($list.Title)'"
                $contentTypesJson = m365 spo list contenttype list --webUrl $SiteUrl --listId $list.Id --output json 2>&1
               if ($LASTEXITCODE -eq 0) {
                    $contentTypes = @($contentTypesJson | ConvertFrom-Json) | Where-Object { -not [string]::IsNullOrEmpty($_.ClientFormCustomFormatter) }

                    foreach ($contentType in $contentTypes) {
                        $folderPath = Join-Path $OutputPath "$($list.Title)\Forms"
                        $fileName = "$($contentType.Name).form-formatter.json"
                        $filePath = Join-Path $folderPath $fileName

                        if ($PSCmdlet.ShouldProcess($filePath, 'Create custom formatter file')) {
                            try {
                                $null = New-Item -Path $folderPath -ItemType Directory -Force
                                $contentType.ClientFormCustomFormatter | ConvertFrom-Json | ConvertTo-Json -Depth 100 | Set-Content -Path $filePath
                                Write-Host "  Exported form: '$($contentType.Name)'" -ForegroundColor Green
                                $script:Summary.FormsExported++
                            }
                            catch {
                                Write-Warning "Failed to export form '$($contentType.Name)': $($_.Exception.Message)"
                                $script:Summary.Failures += [pscustomobject]@{ List = $list.Title; Type = 'Form'; Name = $contentType.Name; Error = $_.Exception.Message }
                            }
                        }
                    }
                }
            }
            catch {
                Write-Warning "Failed to process content types for list '$($list.Title)': $($_.Exception.Message)"
                $script:Summary.Failures += [pscustomobject]@{ List = $list.Title; Type = 'ContentTypes'; Name = 'N/A'; Error = $_.Exception.Message }
            }
        }
    }

    end {
        Write-Host "`n--- Export Summary ---" -ForegroundColor Cyan
        Write-Host "Lists processed     : $($script:Summary.ListsProcessed)"
        Write-Host "Fields exported     : $($script:Summary.FieldsExported)"
        Write-Host "Views exported      : $($script:Summary.ViewsExported)"
        Write-Host "Forms exported      : $($script:Summary.FormsExported)"
        Write-Host "Failures            : $($script:Summary.Failures.Count)"

        if ($script:Summary.Failures.Count -gt 0) {
           Write-Host "`nFailed items:" -ForegroundColor Yellow
           $script:Summary.Failures | Format-Table -AutoSize
       }

        if (-not [string]::IsNullOrEmpty($ExportCsvPath)) {
            Write-Verbose "Exporting results to CSV at '$ExportCsvPath'"
            if ($PSCmdlet.ShouldProcess($ExportCsvPath, 'Export results to CSV')) {
                try {
                    $csvData = @()
                    
                    foreach ($failure in $script:Summary.Failures) {
                        $csvData += [pscustomobject]@{
                            List    = $failure.List
                            Type    = $failure.Type
                            Name    = $failure.Name
                            Status  = 'Failed'
                            Error   = $failure.Error
                        }
                    }
                    
                    if ($csvData.Count -gt 0) {
                        $csvData | Export-Csv -Path $ExportCsvPath -NoTypeInformation -Force
                        Write-Host "Results exported to: $ExportCsvPath" -ForegroundColor Green
                    } else {
                        Write-Host "No failures to export." -ForegroundColor Green
                    }
                }
                catch {
                    Write-Warning "Failed to export CSV: $($_.Exception.Message)"
                }
            }
        }

       return [pscustomobject]$script:Summary
    }
}

# Example usage
Export-SPOCustomFormatting -SiteUrl 'https://contoso.sharepoint.com/sites/project-x' -ExportCsvPath 'ExportResults.csv' -Verbose
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Dan Toft](https://twitter.com/tanddant) |
| Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-export-all-customformatting" aria-hidden="true" />

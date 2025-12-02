

# Generate Demo Events for SharePoint Events List

## Summary

This sample script generates demo events for the SharePoint Events List within a modern site. This uses a CSV to load sample events that includes, titles, descriptions, start, end times, category. The script will use the current date and add days to the events from the CSV file to create example events in the future relative to the current date.

This scenario shows how to bulk add events and could be modified to bulk load events into the SharePoint Events List.

> [!Note]
> The script does not include the imagery shown in the screenshot, I have update stock images as its very quick to do.
> In addition, the location does not resolve the longitude and latitude coordinates.



![Example Screenshot](assets/example.png)

# [PnP PowerShell](#tab/pnpps)

```powershell
Connect-PnPOnline https://contoso.sharepoint.com/sites/demo -Interactive

# Load Data from CSV File
$eventsToAdd = Import-CSV events.csv

# Each time you run the script it will use the current date and set it X number of days in the future. 
$eventsToAdd | ForEach-Object {

    # Uses todays date as the date
    # Date format is Month/Day/Year Hour/Minute
    $format = "MM/dd/yyyy HH:mm"
    
    # Add Days
    $today = ((Get-Date).AddDays($_.FutureDays)).ToString("MM/dd/yyyy")
    $startDateTime = $today + " $($_.StartTime)"
    $endDateTime = $today + " $($_.EndTime)"

    Write-Host "Adding $startDateTime StartDate"
    Write-Host "Adding $endDateTime EndDate"

    $values = @{
        "Title" = $_.Title; 
        "Category" = $_.Category; 
        "Description" = $_.Description; 
        "fAllDayEvent" = $_.AllDayEvent; 
        "Location" = $_.Location;
        "EventDate" = $startDateTime;
        "EndDate" = $endDateTime;
    }
    
    Add-PnPListItem -List "Events" -ContentType "Event" -Values $values
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell
function New-SpoDemoEvents {
    [CmdletBinding(SupportsShouldProcess = $true)]
    param(
        [Parameter(Mandatory = $true, HelpMessage = "SharePoint site URL hosting the Events list")]
        [ValidateNotNullOrEmpty()]
        [string] $SiteUrl,

        [Parameter(HelpMessage = "Name of the target Events list")]
        [ValidateNotNullOrEmpty()]
        [string] $ListTitle = 'Events',

        [Parameter(HelpMessage = "Path to the CSV file with demo events")]
        [ValidateNotNullOrEmpty()]
        [string] $CsvPath = './events.csv',

        [Parameter(HelpMessage = "Skip confirmation prompt before creating list items")]
        [switch] $Force
    )

    begin {
        Write-Verbose "Ensuring CLI authentication"
        $loginOutput = m365 login --ensure 2>&1
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to sign in to CLI for Microsoft 365. CLI output: $loginOutput"
        }

        if (-not (Test-Path -Path $CsvPath -PathType Leaf)) {
            throw "CSV file '$CsvPath' was not found."
        }

        try {
            $events = Import-Csv -Path $CsvPath -ErrorAction Stop
        }
        catch {
            throw "Failed to import CSV '$CsvPath'. Error: $($_.Exception.Message)"
        }

        if ($events.Count -eq 0) {
            throw "CSV '$CsvPath' does not contain any rows."
        }

        $summary = [ordered]@{
            Total   = $events.Count
            Added   = 0
            Skipped = 0
            Errors  = 0
        }

        $culture = [System.Globalization.CultureInfo]::InvariantCulture
    }

    process {
        foreach ($event in $events) {
            try {
                $futureDays = [int]$event.FutureDays
            }
            catch {
                Write-Warning "Invalid FutureDays value '$($event.FutureDays)' for '$($event.Title)'. Skipping."
                $summary.Skipped++
                continue
            }

            $baseDate = (Get-Date).Date.AddDays($futureDays)

            try {
                $startTime = [DateTime]::ParseExact($event.StartTime, 'HH:mm', $culture)
                $endTime = [DateTime]::ParseExact($event.EndTime, 'HH:mm', $culture)
            }
            catch {
                Write-Warning "Failed to parse start or end time for '$($event.Title)'. Expected HH:mm format."
                $summary.Skipped++
                continue
            }

            $startDateTime = $baseDate.AddHours($startTime.Hour).AddMinutes($startTime.Minute)
            $endDateTime = $baseDate.AddHours($endTime.Hour).AddMinutes($endTime.Minute)

            if ($endDateTime -lt $startDateTime) {
                $endDateTime = $endDateTime.AddDays(1)
            }

            $allDay = $false
            if ($event.AllDayEvent) {
                [void][System.Boolean]::TryParse($event.AllDayEvent, [ref]$allDay)
            }

            if (-not $Force -and -not $PSCmdlet.ShouldProcess($event.Title, 'Create SharePoint event')) {
                $summary.Skipped++
                continue
            }

            $args = @(
                'spo', 'listitem', 'add',
                '--contentType', 'Event',
                '--listTitle', $ListTitle,
                '--webUrl', $SiteUrl,
                '--Title', $event.Title,
                '--Category', $event.Category,
                '--Description', $event.Description,
                '--Location', $event.Location,
                '--EventDate', $startDateTime.ToString('s'),
                '--EndDate', $endDateTime.ToString('s'),
                '--fAllDayEvent', ($allDay.ToString().ToLowerInvariant()),
                '--output', 'json'
            )

            $addOutput = m365 @args 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to add event '$($event.Title)'. CLI output: $addOutput"
                $summary.Errors++
                continue
            }

            $summary.Added++
            Write-Host "Added event '$($event.Title)' starting $($startDateTime.ToString('g'))" -ForegroundColor Green
        }
    }

    end {
        Write-Host "Demo event generation summary:" -ForegroundColor Cyan
        Write-Host ("- Total rows processed : {0}" -f $summary.Total)
        Write-Host ("- Added successfully   : {0}" -f $summary.Added)
        Write-Host ("- Skipped              : {0}" -f $summary.Skipped)
        Write-Host ("- Failed               : {0}" -f $summary.Errors)
    }
}

# Example usage
New-SpoDemoEvents -SiteUrl "https://contoso.sharepoint.com/sites/demo" -CsvPath "./events.csv" -Verbose
```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
***

# [CSV](#tab/csv)

```

Title,FutureDays,StartTime,EndTime,Description,Category,AllDayEvent,Location
All Company Away Day,10,10:00,11:00,This is a corporate event for everyone to join,Corporate Event,FALSE,"Bath, England"
Senior Management Quarterly Announcements,90,12:00,13:00,This is a corporate event for everyone to join,Corporate Event,FALSE,"Bath, England"
Senior Management Quarterly Announcements,180,10:00,11:00,This is a corporate event for everyone to join,Corporate Event,FALSE,"Bristol, England"
Company Internal Conference,35,09:00,17:00,This is a corporate event for everyone to join,Corporate Event,FALSE,"Cardiff, Wales"
Training Event: Office 365,15,10:00,11:00,This is a corporate event for everyone to join,Corporate Event,FALSE,"London, England"
Training Event: Security,20,12:00,13:00,This is a corporate event for everyone to join,Corporate Event,FALSE,"London, England"
Training Event: Using Teams,25,12:00,13:00,This is a corporate event for everyone to join,Corporate Event,FALSE,"London, England"

```

> [!Note]
> Save the CSV block of text as a CSV file and name it "events.csv"

***

## Contributors

| Author(s) |
|-----------|
| Paul Bullock |
| Adam Wójcik |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-generate-demo-events" aria-hidden="true" />

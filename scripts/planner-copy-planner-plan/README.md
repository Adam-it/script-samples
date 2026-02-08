

# Copy Planner plan

## Summary

With this sample, you can copy an existing Planner plan to a specific group using PnP PowerShell or CLI for Microsoft 365. This script will create a new plan with the same name and copy all buckets and tasks.

Following data will be copied:
* Plan name
* Buckets
* Tasks
  * Title
  * Notes
  * Progress
  * Priority
  * Start date
  * Due date

![Example Screenshot](assets/example.png)

## Script parameters

| Parameter | Mandatory | Description |
| --- | --- | --- |
| SourcePlanId | Yes | Source Planner plan to copy. |
| DestinationGroupId | Yes | Destination group ID to copy the plan to. |

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding()]
param (
  [Parameter(Mandatory = $true, HelpMessage = "ID of the source Planner plan to copy.")]
  [ValidateNotNullOrEmpty()]
  [string]$SourcePlanId,

  [Parameter(Mandatory = $true, HelpMessage = "ID of the destination group to copy the plan to.")]
  [ValidateNotNullOrEmpty()]
  [guid]$DestinationGroupId
)
begin {
  $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
  Start-Transcript -Path "cli-copy-planner-plan-log-$timestamp.log"

  Write-Host "Connecting to Microsoft 365..." -ForegroundColor Yellow
  m365 login --ensure
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to authenticate to Microsoft 365"
  }

  $script:Summary = @{
    BucketsCreated = 0
    TasksCreated = 0
    Failures = 0
  }
}
process {
  Write-Host "Copying plan..." -ForegroundColor Yellow
  $ProgressActivity = "Copying Planner plan"
  Write-Progress -Activity $ProgressActivity -Status "Reading source plan data..." -PercentComplete 0

  $plan = m365 planner plan get --id $SourcePlanId --output json | ConvertFrom-Json
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to retrieve source plan with ID: $SourcePlanId"
  }

  $buckets = m365 planner bucket list --planId $SourcePlanId --output json | ConvertFrom-Json
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to retrieve buckets for plan: $SourcePlanId"
  }

  $tasks = m365 planner task list --planId $SourcePlanId --output json | ConvertFrom-Json
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to retrieve tasks for plan: $SourcePlanId"
  }

  # Buckets and tasks are fetched in reverse order
  [array]::Reverse($buckets)
  [array]::Reverse($tasks)

  Write-Host "Found $($buckets.Count) buckets and $($tasks.Count) tasks to copy" -ForegroundColor Cyan
  Write-Progress -Activity $ProgressActivity -Status "Creating new plan '$($plan.Title)' at destination group..." -PercentComplete 20

  $clonedPlan = m365 planner plan add --ownerGroupId $DestinationGroupId --title "$($plan.Title)" --output json | ConvertFrom-Json
  if ($LASTEXITCODE -ne 0) {
    throw "Failed to create plan in destination group: $DestinationGroupId"
  }

  Write-Host "Plan created successfully: $($clonedPlan.id)" -ForegroundColor Green
  Write-Progress -Activity $ProgressActivity -Status "Creating $($buckets.Count) buckets..." -PercentComplete 40

  $bucketMapping = @{ }
  $bucketCounter = 0
  foreach ($bucket in $buckets) {
    $bucketCounter++
    Write-Progress -Activity $ProgressActivity -Status "Creating bucket $bucketCounter/$($buckets.Count): $($bucket.name)" -PercentComplete (40 + ($bucketCounter / $buckets.Count * 20))

    try {
      $clonedBucket = m365 planner bucket add --planId $clonedPlan.id --name "$($bucket.name)" --output json | ConvertFrom-Json
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to create bucket: $($bucket.name)"
        $script:Summary.Failures++
        continue
      }
      $bucketMapping[$bucket.id] = $clonedBucket.id
      $script:Summary.BucketsCreated++
    }
    catch {
      Write-Warning "Error creating bucket '$($bucket.name)': $_"
      $script:Summary.Failures++
      continue
    }
  }

  Write-Progress -Activity $ProgressActivity -Status "Creating $($tasks.Count) tasks..." -PercentComplete 60

  $taskCounter = 0
  foreach ($task in $tasks) {
    $taskCounter++
    Write-Progress -Activity $ProgressActivity -Status "Creating task $taskCounter/$($tasks.Count): $($task.title)" -PercentComplete (60 + ($taskCounter / $tasks.Count * 35))

    try {
      $args = @(
        'planner', 'task', 'add',
        '--planId', $clonedPlan.id,
        '--bucketId', $bucketMapping[$task.bucketId],
        '--title', $task.title,
        '--percentComplete', $task.percentComplete,
        '--priority', $task.priority,
        '--output', 'json'
      )

      if ($task.hasDescription) {
        $details = m365 planner task get --id $task.id --output json | ConvertFrom-Json
        if ($LASTEXITCODE -ne 0) {
          Write-Warning "Failed to get task details for task '$($task.title)' (ID: $($task.id))"
          $script:Summary.Failures++
          continue
        }
        if ($details.description) {
          $args += '--description', $details.description
        }
      }
      if ($null -ne $task.startDateTime) {
        $args += '--startDateTime', $task.startDateTime
      }
      if ($null -ne $task.dueDateTime) {
        $args += '--dueDateTime', $task.dueDateTime
      }

      $result = m365 @args
      if ($LASTEXITCODE -ne 0) {
        Write-Warning "Failed to create task: $($task.title)"
        $script:Summary.Failures++
        continue
      }
      $script:Summary.TasksCreated++
    }
    catch {
      Write-Warning "Error creating task '$($task.title)': $_"
      $script:Summary.Failures++
      continue
    }
  }
}
end {
  Write-Progress -Activity $ProgressActivity -Status "Plan copied!" -PercentComplete 100 -Completed

  Write-Host "`n========== Summary ==========" -ForegroundColor Cyan
  Write-Host "Source Plan: $($plan.Title) (ID: $SourcePlanId)" -ForegroundColor Green
  Write-Host "Destination Plan: $($clonedPlan.id)" -ForegroundColor Green
  Write-Host "Buckets Created: $($Summary.BucketsCreated)" -ForegroundColor Green
  Write-Host "Tasks Created: $($Summary.TasksCreated)" -ForegroundColor Green
  Write-Host "Failures: $($Summary.Failures)" -ForegroundColor $(if ($Summary.Failures -gt 0) { 'Red' } else { 'Green' })
  Write-Host "==============================`n" -ForegroundColor Cyan

  Stop-Transcript
}

# Example 1: Copy plan to a group
# .\Copy-Planner-plan.ps1 -SourcePlanId "xqQg5FS2LkCp935s-FIFm2QAFkHM" -DestinationGroupId "00000000-0000-0000-0000-000000000000"

# Example 2: Copy plan with verbose logging
# .\Copy-Planner-plan.ps1 -SourcePlanId "abc123def456" -DestinationGroupId "11111111-1111-1111-1111-111111111111" -Verbose

# Example 3: Copy plan to different group
# .\Copy-Planner-plan.ps1 -SourcePlanId "plan-guid-here" -DestinationGroupId "22222222-2222-2222-2222-222222222222"

# Example 4: Copy plan with error handling (check transcript log for details)
# .\Copy-Planner-plan.ps1 -SourcePlanId "xyz789" -DestinationGroupId "33333333-3333-3333-3333-333333333333"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

# [PnP PowerShell](#tab/pnpps)
```powershell
# Usage example:
# .\Copy-Planner-plan.ps1 -SourcePlanId 73RAujmtbEKfwfA20Sb46ZgAB_Vi -DestinationGroupId 00000000-0000-0000-0000-000000000000 -AdminUrl https://contoso-admin.sharepoint.com

[CmdletBinding()]
param (
  [Parameter(Mandatory = $true, HelpMessage = "Source Planner plan to copy e.g. xqQg5FS2LkCp935s-FIFm2QAFkHM.")]
  [string]$SourcePlanId,

  [Parameter(Mandatory = $true, HelpMessage = "Destination group ID to copy the plan to e.g. 00000000-0000-0000-0000-000000000001.")]
  [string]$DestinationGroupId,

  [Parameter(Mandatory = $true, HelpMessage = "The Url of the SharePoint Admin Center, e.g.https://contoso-admin.sharepoint.com  ")]
  [string]$AdminUrl
)

Begin  {
    Connect-PnPOnline -Url $AdminUrl -Interactive
}

process {
  Write-Host "Copying plan..." -ForegroundColor Yellow
  $ProgressActivity = "Copying Planner plan"
  Write-Progress -Activity $ProgressActivity -Status "Reading source plan data" -PercentComplete 0

  $plan = Get-PnPPlannerPlan -Id $SourcePlanId
  $buckets = Get-PnPPlannerBucket -PlanId $SourcePlanId 
  $tasks = Get-PnPPlannerTask -PlanId $SourcePlanId 

  # Buckets and tasks are fetched in reverse order
  [array]::Reverse($buckets)
  [array]::Reverse($tasks)

  Write-Progress -Activity $ProgressActivity -Status "Creating new plan at destination group" -PercentComplete 25

  $clonedPlan = New-PnPPlannerPlan -Group $DestinationGroupId -Title $plan.Title

  Write-Progress -Activity $ProgressActivity -Status "Creating buckets" -PercentComplete 50

  # Create mapping object for buckets {Key: sourceId; Value: destinationId}
  $bucketMapping = @{}
  foreach ($bucket in $buckets) {
    $clonedBucket = Add-PnPPlannerBucket -PlanId $clonedPlan.id -Name $bucket.name 
    $bucketMapping[$bucket.id] = $clonedBucket.id
  }

  Write-Progress -Activity $ProgressActivity -Status "Creating tasks" -PercentComplete 75

  foreach ($task in $tasks) {
    $command = "Add-PnPPlannerTask -planId $($clonedPlan.id) -bucket $($bucketMapping[$task.bucketId]) -Title '$($task.title.Replace("'", "''"))' -PercentComplete $($task.percentComplete)  -Priority $($task.priority)"
    
    # Append optional options when needed
    if ($task.hasDescription) {
      $details = Get-PnPPlannerTask -TaskId $task.id
      $command += " -Description '$($details.description.Replace("'", "''"))'"
    }
    if ($null -ne $task.startDateTime) {
      $command += " -StartDateTime '"+ $(get-date -date $task.StartDateTime -Format 'dd/MM/yyyy HH:mm') + "'"
    }
    if ($null -ne $task.dueDateTime) {
      $command += " -DueDateTime '" + $(get-date -date $task.dueDateTime -Format 'dd/MM/yyyy HH:mm') +"'"
    }

    Invoke-Expression $command | Out-Null
  }
}
end {
  Write-Progress -Activity $ProgressActivity -Status "Plan copied!" -PercentComplete 100 -Completed
  Write-Host "Script completed!" -ForegroundColor Green
}

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***
## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| Milan Holemans |
| [Reshmee Auckloo](https://github.com/reshmee011)|

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/planner-copy-planner-plan" aria-hidden="true" />

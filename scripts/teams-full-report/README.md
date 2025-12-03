

      
# Teams Full Report
![Outputs](assets/header.png)
## Summary

Script to generate Team's full report, gathering all Teams,Channels,Tabs available info.

It includes :
* **Teams**  :  

    Visibility, Url, Classification, CreatedDate, DeletedDate,  
    Mail, MailEnabled, MailNickname, RenewedDate, SecurityEnabled,  
    SecurityIdentified, Theme , Owners, Members, Guests

* **Channels** :  

    PrimaryChannel, Created Date, Email, IdIsFavoriteByDefault,  
    MembershipType, ModerationSettings, Url

* **Tabs**:  

    PrimaryChannel, Created Date, Email, IdIsFavoriteByDefault,  
    MembershipType, ModerationSettings, Url,  
    TabDisplayName,TabWebUrl, TeamsAppId
  


This script is a subset of the SPO powershell packages with content (PnPCandy) concept already been used across many projects.  


Excelsior, hum? :P  

# [PnP PowerShell](#tab/pnpps)

```powershell

[CmdletBinding()]
param (
    [Parameter(Mandatory = $True)]
    [string]$Tenant ,
    [Parameter(Mandatory = $False)]
    [string]$Team,
    [Parameter(Mandatory = $False)]
    [string]$ExportPath= ".\"
)
begin {
    $ErrorActionPreference = "Stop"
    Import-Module PnP.PowerShell   
    
    function Capitalize($objMain) {
        if ($null -ne $objMain) {

            #test if its an array of objects
            $objMain.foreach({
                    $obj = $_
                    $members = $obj | Get-Member -MemberType NoteProperty
                    $members | ForEach-Object {
           
                        $name = [regex]::replace($_.Name, '(^|_)(.)', { $args[0].Groups[2].Value.ToUpper() })
                        $value = $obj.PSObject.Properties[$_.Name].value
                        $obj.PSObject.Properties.Remove($_.Name)
                        # add a new NoteProperty 'fqdn'
                        $obj | Add-Member -Name $name  -Value $value -MemberType NoteProperty -Force 
                        $added = $obj.PSObject.Properties[$name]
    
                        if ($added.TypeNameOfValue.ToLower() -like "*pscustom*") {
                            $obj."$($_.Name)" = Capitalize -obj $obj."$($_.Name)"
                        }
                    }
                })
           
        }
        $objMain
    }
    function Get-UserInfo($obj)
    {
        $Info=" "
      
        $obj |Select-Object @{n = 'Users'; e = {'[' +  $_.DisplayName + ':' + $_.UserPrincipalName +']' } } | ForEach-Object { $Info += $_.Users +";"  }
        $Info= $Info.Substring(0,$Info.Length-1).Trim()
        $Info
    }
    $msg = "`n`r

    █▀█ █▄░█ █▀█ █▀▀ ▄▀█ █▄░█ █▀▄ █▄█
    █▀▀ █░▀█ █▀▀ █▄▄ █▀█ █░▀█ █▄▀ ░█░  `n    MSTeamsToolSet: `n`r    Teams full report    `n`n    ...aka ... [teams-full-report]
    `n"

    $msg += ('#' * 70) + "`n"
    Write-Output  $msg
    
    ## Validate if Tenant value is ok
    if ($Tenant -notmatch '.onmicrosoft.com') {
        $msg = "Provided Tenant is not valid. Please use the following format [Tenant].onmicrosoft.com. Example:pnpcady.onmicrosoft.com"
        throw $msg
    }
    $tenantPrefix = $Tenant.ToLower().Replace(".onmicrosoft.com", "")
    $url = "https://$tenantPrefix-admin.sharepoint.com"

    Write-Output "Connecting to $Url"
    Connect-PnPOnline -Url $url -Interactive -Tenant $Tenant
    $accesstoken =  Get-PnPAccessToken
 
}
process {
   
    Write-Output " Getting Team(s) $Team"
    $listOfTeams = Get-PnPMicrosoft365Group  -IncludeSiteUrl | Where-object { $_.HasTeam }

    if (($null -ne $Team) -and ($Team -ne ""))
    {
        $listOfTeams =  $listOfTeams | Where-object {($_.id -eq $Team) -or ($_.Displayname -eq $Team)}
    }
    
    if($null -ne $listOfTeams)
    {
        Write-Output " [$($listOfTeams.Length)] Team(s)"
    }
    else {
        Write-Output " No Team(s) found"
    }
   
                                                       
    $list = @()  
    $listOfTeams | ForEach-Object {
        $tm = $_

        Write-Output "  Team:$($tm.DisplayName)"

        Write-Output "   Get membership (Onwers,Members,Guests)" 
        $Owners = Get-PnPMicrosoft365GroupOwners -Identity $tm.GroupId
        $OwnersInfo= Get-UserInfo -obj $Owners

        $Members = Get-PnPMicrosoft365GroupMembers -Identity $tm.GroupId
        $MembersInfo= Get-UserInfo -obj $Members 
        
        $Guests =  Get-PnPTeamsUser -Team $tm.DisplayName  -Role Guest
        $GuestsInfo= Get-UserInfo -obj $Guests 

        $tm | Add-Member -Name "Owners" -MemberType NoteProperty -Value $Owners  -Force
        $tm | Add-Member -Name "OwnersInfo" -MemberType NoteProperty -Value $OwnersInfo  -Force
        $tm | Add-Member -Name "Members" -MemberType NoteProperty -Value $Members  -Force
        $tm | Add-Member -Name "MembersInfo" -MemberType NoteProperty -Value $MembersInfo  -Force
        $tm | Add-Member -Name "Guest" -MemberType NoteProperty -Value $Guests  -Force    
        $tm | Add-Member -Name "GuestInfo" -MemberType NoteProperty -Value $GuestsInfo  -Force 

        Write-Output "   Membership (Onwers,Members,Guests) collected ! " 
        $Body = @{
            "Resource"      = "https://graph.microsoft.com"
        }
        
        #get all channels
        Write-Output "   Getting Channels"
        $url = "https://graph.microsoft.com/beta/teams/$($tm.Id)/channels"  
        $allChannels = @((Invoke-RestMethod -Uri $url -Headers @{Authorization = "Bearer $accesstoken"; "Content-Type" = "application/json" ; "Resource"      = "https://graph.microsoft.com"}  -Method Get).value) 
        $allChannels = Capitalize -obj $allChannels
       
        Write-Output "    Get Primary Channel"
        $url = "https://graph.microsoft.com/v1.0/teams/$($tm.Id)/primaryChannel"  
        $primaryChannel = Invoke-RestMethod -Uri $url -Headers @{Authorization = "Bearer $accesstoken"; "Content-Type" = "application/json" } -Body $Body -Method Get
      
        $allChannels | ForEach-object {   
            [PsObject] $chn = [PsObject]  $_
            # $channel
            Write-Output ("    [" + $chn.DisplayName + "] Getting Tabs")
            $isPrimaryChannel = ($primaryChannel.id -eq $chn.Id)
            #Add PrimaryChanell boolean field to each channel
            $chn | Add-Member -Name "PrimaryChannel" -MemberType NoteProperty -Value $isPrimaryChannel  
            $url = "https://graph.microsoft.com/v1.0/teams/$($tm.Id)/channels/" + $chn.Id + "/tabs?`$expand=teamsApp"  
            $tabs = Invoke-RestMethod -Uri $url -Headers @{Authorization = "Bearer $accesstoken"; "Content-Type" = "application/json" } -Method Get
            $tabs = Capitalize -obj $tabs.Value
            $chn | Add-Member -Name "Tabs" -MemberType NoteProperty -Value  $tabs  -Force
              Write-Output ("    [" + $chn.DisplayName + "] Tabs collected !")

        }
        Write-Output "   All Channels collected!"
        Write-Output ("  Getting Team ownership")
        $teamOwners = Get-PnPTeamsUser -Team $tm.DisplayName -Role Owner
        $teamMembers = Get-PnPTeamsUser -Team $tm.DisplayName -Role Member
        $teamGuest = Get-PnPTeamsUser -Team $tm.DisplayName -Role Guest
        $tm | Add-Member -Name "Channels" -MemberType NoteProperty -Value $allChannels -Force
        $tm | Add-Member -Name "Owners" -MemberType NoteProperty -Value $teamOwners  -Force
        $tm | Add-Member -Name "Members" -MemberType NoteProperty -Value $teamMembers  -Force
        $tm | Add-Member -Name "Guest" -MemberType NoteProperty -Value $teamGuest  -Force
    }
    Disconnect-PnPOnline
    Write-Output "Disconnected"

    $exportTeams = $listOfTeams |  Sort-Object Id
   
    $teams = $exportTeams |Select-Object @{n = 'TeamId'; e = { $_.Id } } ,  @{n = 'TeamDisplayName'; e = { $_.DisplayName } } ,  @{n = 'TeamDescription'; e = { $_.Description } },  `
    Visibility, SiteUrl, Classification, CreatedDateTime, DeletedDateTime, `
    Mail, MailEnabled, MailNickname, RenewedDateTime, SecurityEnabled, SecurityIdentified, Theme , OwnersInfo, MembersInfo, GuestsInfo 
    
    $teamsChannels = $exportTeams | Select-Object @{n = 'TeamId'; e = { $_.Id } }, @{n = 'TeamDisplayName'; e = { $_.DisplayName } } , @{n = 'TeamDescription'; e = { $_.Description } } -ExpandProperty Channels| Select-Object $_.Channels  
    $teamsChannels = $teamsChannels| Select-Object TeamDisplayName,  @{n = 'ChannelId'; e = { $_.Id } } , @{n = 'ChannelDisplayName'; e = { $_.DisplayName } } ,  @{n = 'ChannelDescription'; e = { $_.Description } }  , `
                                     PrimaryChannel, CreatedDateTime, Email, IdIsFavoriteByDefault, MembershipType, ModerationSettings, WebUrl, Tabs

    $teamsChannelsTabs =$teamsChannels | Select-Object  TeamDisplayName, ChannelDisplayName  -ExpandProperty Tabs
    $teamsChannelsTabs =$teamsChannelsTabs | Select-Object  TeamDisplayName,ChannelDisplayName,@{n = 'TabId'; e = { $_.Id } },@{n = 'TabDisplayName'; e = { $_.DisplayName } }  , @{n = 'TabWebUrl'; e = { $_.WebUrl } }  -ExpandProperty TeamsApp
    $teamsChannelsTabs =$teamsChannelsTabs | Select-Object  TeamDisplayName,ChannelDisplayName,TabId, TabDisplayName,TabWebUrl, @{n = 'TeamsAppId'; e = { $_.Id } } 

    $teamsChannels = $teamsChannels | Select-Object TeamDisplayName,ChannelId,	ChannelDisplayName,	ChannelDescription,	PrimaryChannel,	CreatedDateTime, Email,	IdIsFavoriteByDefault,	MembershipType,	 ModerationSettings,WebUrl
    Write-Output "Export all Teams info"
    $path= (Resolve-path -Path $ExportPath).Path
    $teams |  Export-Csv -Path "$path\Teams.csv" -Force
    $teamsChannels |Export-Csv -Path "$path\TeamsChannels.csv" -Force
    $teamsChannelsTabs |  Export-Csv -Path "$path\TeamsChannelsTabs.csv" -Force
    Write-Output "All Teams info exported at [$path] "

}


```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)
```powershell

[CmdletBinding()]
param (
    [Parameter(Mandatory = $True, HelpMessage = 'Tenant primary domain (e.g., contoso.onmicrosoft.com).')]
    [string]$Tenant,
    [Parameter(Mandatory = $False, HelpMessage = 'Optional team display name or ID to scope the report.')] 
    [string]$Team,
    [Parameter(Mandatory = $False, HelpMessage = 'Folder where the CSV reports will be saved. Defaults to current directory.')] 
    [string]$ExportPath= ".\"
)
begin {
    $ErrorActionPreference = "Stop"   
    
    function Capitalize($objMain) {
        if ($null -eq $objMain) {
            return $null
        }

        # test if its an array of objects
        $objMain.foreach({
                $obj = $_
                $members = $obj | Get-Member -MemberType NoteProperty
                $members | ForEach-Object {
                    $name = [regex]::replace($_.Name, '(^|_)(.)', { $args[0].Groups[2].Value.ToUpper() })
                    $value = $obj.PSObject.Properties[$_.Name].value
                    $obj.PSObject.Properties.Remove($_.Name)
                    # add a new NoteProperty 'fqdn'
                    $obj | Add-Member -Name $name  -Value $value -MemberType NoteProperty -Force 
                    $added = $obj.PSObject.Properties[$name]
                }
            })

        $objMain
    }

    function Get-UserInfo($obj)
    {
        $Info = " "
      
        $obj | Select-Object @{n = 'Users'; e = {'[' +  $_.DisplayName + ':' + $_.UserPrincipalName +']' } } | ForEach-Object { $Info += $_.Users +";"  }
        $Info = $Info.Substring(0, $Info.Length-1).Trim()
        $Info
    }
    
    ## Validate if Tenant value is ok
    if ($Tenant -notmatch '.onmicrosoft.com')
        throw "Provided Tenant is not valid. Please use the following format [Tenant].onmicrosoft.com. Example:pnpcady.onmicrosoft.com"

    $tenantPrefix = $Tenant.ToLower().Replace(".onmicrosoft.com", "")
    $url = "https://$tenantPrefix-admin.sharepoint.com"
    $loginOutput = m365 login --ensure 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to authenticate to Microsoft 365. CLI output: $loginOutput"
    }
}

process {
    Write-Verbose "Getting Team(s) $Team"
    $teamsResponse = m365 teams team list --output json 2>&1
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to retrieve teams. CLI output: $teamsResponse"
    }

    try {
        $listOfTeams = $teamsResponse | ConvertFrom-Json
    }
    catch {
        throw "Unable to parse teams response as JSON. Error: $($_.Exception.Message)"
    }

    if (($null -ne $Team) -and ($Team -ne ""))
    {
        $listOfTeams =  $listOfTeams | Where-object {($_.id -eq $Team) -or ($_.Displayname -eq $Team)}
    }
    
    if($null -ne $listOfTeams)
    {
        $teamCount = $listOfTeams.Count
        Write-Verbose "Processing $teamCount teams..."
    }
    else {
        Write-Output " No Team(s) found"
    }
                                                
    $list = @()  
    $listOfTeams | ForEach-Object {
        $tm = $_

        Write-Verbose "Processing team '$($tm.displayName)'"

        Write-Verbose "Retrieving group details"
        $groupResponse = m365 entra m365group get --id $tm.id --withSiteUrl --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to retrieve group details for team '$($tm.displayName)'. CLI output: $groupResponse"
            return
        }

        try {
            $Group = $groupResponse | ConvertFrom-Json
        }
        catch {
            Write-Warning "Unable to parse group details for team '$($tm.displayName)'. Error: $($_.Exception.Message)"
            return
        }
 
        $tm | Add-Member -Name "Visibility" -MemberType NoteProperty -Value $Group.Visibility  -Force
        $tm | Add-Member -Name "SiteUrl" -MemberType NoteProperty -Value $Group.SiteUrl  -Force
        $tm | Add-Member -Name "Classification" -MemberType NoteProperty -Value $Group.Classification  -Force
        $tm | Add-Member -Name "CreatedDateTime" -MemberType NoteProperty -Value $Group.CreatedDateTime  -Force
        $tm | Add-Member -Name "DeletedDateTime" -MemberType NoteProperty -Value $Group.DeletedDateTime  -Force
        $tm | Add-Member -Name "Mail" -MemberType NoteProperty -Value $Group.Mail  -Force
        $tm | Add-Member -Name "MailEnabled" -MemberType NoteProperty -Value $Group.MailEnabled  -Force
        $tm | Add-Member -Name "MailNickname" -MemberType NoteProperty -Value $Group.MailNickname  -Force
        $tm | Add-Member -Name "RenewedDateTime" -MemberType NoteProperty -Value $Group.RenewedDateTime  -Force
        $tm | Add-Member -Name "SecurityEnabled" -MemberType NoteProperty -Value $Group.SecurityEnabled  -Force
        $tm | Add-Member -Name "SecurityIdentified" -MemberType NoteProperty -Value $Group.SecurityIdentified  -Force
        $tm | Add-Member -Name "Theme" -MemberType NoteProperty -Value $Group.Theme  -Force
       
        Write-Verbose "Retrieving membership (Owners, Members, Guests)"
        $Owners = @()
        $Members = @()
        $Guests = @()

        $ownersResponse = m365 teams user list --teamId $tm.id --role Owner --output json 2>&1
        $ownersExitCode = $LASTEXITCODE
        $membersResponse = m365 teams user list --teamId $tm.id --role Member --output json 2>&1
        $membersExitCode = $LASTEXITCODE
        $guestsResponse = m365 teams user list --teamId $tm.id --role Guest --output json 2>&1
        $guestsExitCode = $LASTEXITCODE

        try {
            if ($ownersExitCode -eq 0 -and $ownersResponse) {
                $Owners = $ownersResponse | ConvertFrom-Json
            }
            elseif ($ownersExitCode -ne 0) {
                Write-Warning "Failed to retrieve owners for team '$($tm.displayName)'. CLI output: $ownersResponse"
            }

            if ($membersExitCode -eq 0 -and $membersResponse) {
                $Members = $membersResponse | ConvertFrom-Json
            }
            elseif ($membersExitCode -ne 0) {
                Write-Warning "Failed to retrieve members for team '$($tm.displayName)'. CLI output: $membersResponse"
            }

            if ($guestsExitCode -eq 0 -and $guestsResponse) {
                $Guests = $guestsResponse | ConvertFrom-Json
            }
            elseif ($guestsExitCode -ne 0) {
                Write-Warning "Failed to retrieve guests for team '$($tm.displayName)'. CLI output: $guestsResponse"
            }
        }
        catch {
            Write-Warning "Unable to parse membership data for team '$($tm.displayName)'. Error: $($_.Exception.Message)"
        }

        $OwnersInfo= Get-UserInfo -obj $Owners
        $MembersInfo= Get-UserInfo -obj $Members 
        $GuestsInfo= Get-UserInfo -obj $Guests 

        $tm | Add-Member -Name "Owners" -MemberType NoteProperty -Value $Owners  -Force
        $tm | Add-Member -Name "OwnersInfo" -MemberType NoteProperty -Value $OwnersInfo  -Force
        $tm | Add-Member -Name "Members" -MemberType NoteProperty -Value $Members  -Force
        $tm | Add-Member -Name "MembersInfo" -MemberType NoteProperty -Value $MembersInfo  -Force
        $tm | Add-Member -Name "Guest" -MemberType NoteProperty -Value $Guests  -Force    
        $tm | Add-Member -Name "GuestInfo" -MemberType NoteProperty -Value $GuestsInfo  -Force 

        Write-Verbose "Membership (Owners, Members, Guests) collected"
        
        #get all channels
        Write-Verbose "Retrieving channels"
        $channelsResponse = m365 teams channel list --teamId $tm.id --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to retrieve channels for team '$($tm.displayName)'. CLI output: $channelsResponse"
            return
        }

        try {
            $allChannels = $channelsResponse | ConvertFrom-Json
        }
        catch {
            Write-Warning "Unable to parse channels for team '$($tm.displayName)'. Error: $($_.Exception.Message)"
            return
        }
        $allChannels = Capitalize -obj $allChannels
        
        Write-Verbose "Retrieving primary channel"
        $primaryChannelResponse = m365 teams channel get --teamId $tm.id --primary --output json 2>&1
        if ($LASTEXITCODE -ne 0) {
            Write-Warning "Failed to retrieve primary channel for team '$($tm.displayName)'. CLI output: $primaryChannelResponse"
            $primaryChannel = $null
        }
        else {
            try {
                $primaryChannel = $primaryChannelResponse | ConvertFrom-Json
            }
            catch {
                Write-Warning "Unable to parse primary channel for team '$($tm.displayName)'. Error: $($_.Exception.Message)"
                $primaryChannel = $null
            }
        }
        $allChannels | ForEach-object {   
            [PsObject] $chn = [PsObject]  $_
    
            Write-Verbose ("Retrieving tabs for channel '$($chn.DisplayName)'")
            $isPrimaryChannel = ($primaryChannel.id -eq $chn.Id)
            $tabsResponse = m365 teams tab list --teamId $tm.id --channelId $chn.Id --output json 2>&1
            if ($LASTEXITCODE -ne 0) {
                Write-Warning "Failed to retrieve tabs for team '$($tm.displayName)' channel '$($chn.DisplayName)'. CLI output: $tabsResponse"
                return
            }

            try {
                $tabs = $tabsResponse | ConvertFrom-Json
                $tabs = Capitalize -obj $tabs
            }
            catch {
                Write-Warning "Unable to parse tabs for team '$($tm.displayName)' channel '$($chn.DisplayName)'. Error: $($_.Exception.Message)"
                return
            }
            $chn | Add-Member -Name "Tabs" -MemberType NoteProperty -Value  $tabs  -Force
            Write-Verbose ("Tabs collected for channel '$($chn.DisplayName)'")

        }
        Write-Verbose "All channels collected"
        
        $tm | Add-Member -Name "Channels" -MemberType NoteProperty -Value $allChannels -Force
    }
}

end {
    $exportTeams = $listOfTeams |  Sort-Object Id
   
    $teams = $exportTeams | Select-Object @{n = 'TeamId'; e = { $_.Id } } ,  @{n = 'TeamDisplayName'; e = { $_.DisplayName } } ,  @{n = 'TeamDescription'; e = { $_.Description } },  `
    Visibility, SiteUrl, Classification, CreatedDateTime, DeletedDateTime, `
    Mail, MailEnabled, MailNickname, RenewedDateTime, SecurityEnabled, SecurityIdentified, Theme , OwnersInfo, MembersInfo, GuestsInfo 
    
    $teamsChannels = $exportTeams | Select-Object @{n = 'TeamId'; e = { $_.Id } }, @{n = 'TeamDisplayName'; e = { $_.DisplayName } } , @{n = 'TeamDescription'; e = { $_.Description } } -ExpandProperty Channels| Select-Object $_.Channels  
    $teamsChannels = $teamsChannels| Select-Object TeamDisplayName,  @{n = 'ChannelId'; e = { $_.Id } } , @{n = 'ChannelDisplayName'; e = { $_.DisplayName } } ,  @{n = 'ChannelDescription'; e = { $_.Description } }  , `
                                      CreatedDateTime, Email, IdIsFavoriteByDefault, MembershipType, ModerationSettings, WebUrl, Tabs

    $teamsChannelsTabs = $teamsChannels | Select-Object  TeamDisplayName, ChannelDisplayName - ExpandProperty Tabs
    $teamsChannelsTabs = $teamsChannelsTabs | Select-Object  TeamDisplayName,ChannelDisplayName,@{n = 'TabId'; e = { $_.Id } },@{n = 'TabDisplayName'; e = { $_.DisplayName } }  , @{n = 'TabWebUrl'; e = { $_.WebUrl } }  - ExpandProperty TeamsApp
    $teamsChannelsTabs =$teamsChannelsTabs | Select-Object  TeamDisplayName,ChannelDisplayName,TabId, TabDisplayName,TabWebUrl, @{n = 'TeamsAppId'; e = { $_.Id } } 

    $teamsChannels = $teamsChannels | Select-Object TeamDisplayName,ChannelId,	ChannelDisplayName,	ChannelDescription,	CreatedDateTime, Email,	IdIsFavoriteByDefault,	MembershipType,	 ModerationSettings,WebUrl
    Write-Output "Export all Teams info"
    $path= (Resolve-path -Path $ExportPath).Path
    $teams | Export-Csv -Path "$path\Teams.csv" -Force
    $teamsChannels | Export-Csv -Path "$path\TeamsChannels.csv" -Force
    $teamsChannelsTabs | Export-Csv -Path "$path\TeamsChannelsTabs.csv" -Force
    Write-Output "All Teams info exported at [$path] "
}

# Example usage:
# .\TeamsFullReport.ps1 -Tenant "contoso.onmicrosoft.com" -ExportPath "C:\\Reports" -Verbose

```
[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| [Reshmee Auckloo](https://github.com/reshmee011)|
| Rodrigo Pinto |
| [Adam Wójcik](https://github.com/Adam-it) |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/teams-full-report" aria-hidden="true" />



# Set Page Author Byline

## Summary

Modern pages have a field called `_AuthorByLine`. When we set the value of this field of a page to the login name (or the email address) of a user, we will see the user details appear in the page header byline. However, as soon as we edit the page those details disappear.

So, to fix that, along with the `_AuthorByLine` field, we also need to set the `LayoutWebpartsContent` field of the page with the details of the user.

This script sets the `Authors` and `AuthorByline` properties of the `PageHeader` which in turn set the `_AuthorByLine` field and update the `LayoutWebpartsContent` field of the page.

![Example Screenshot](assets/example.png)

# [PnP PowerShell](#tab/pnpps)

```powershell

    Param(
        [Parameter(mandatory = $true)]
        [string]$SiteUrl,
        [Parameter(mandatory = $true)]
        [string]$PageName,
        [Parameter(mandatory = $true)]
        [string]$UserEmail
    )

    function Set-PageAuthorByline {

        # Connect to the site
        Connect-PnPOnline -Url $SiteUrl;

        # If there is an error in the connection then return
        if ($null -eq $(Get-PnPConnection).ConnectionType) {
            return;
        }

        # Get the page object from the specified page name / url 
        $page = Get-PnPPage -Identity $PageName;

        # Return if page is not found
        if ($null -eq $page) {
            Write-Error "Page Name is not valid";
            return;
        }

        # Get the required user from the User Information list
        Write-Host "Getting user information from User Information list..." -ForegroundColor Yellow;
        $user = Get-PnPUser | Where-Object Email -eq $UserEmail;

        if ($null -ne $user) {
            Write-Host "Got user information from User Information list." -ForegroundColor Yellow;
        }
        else {
            # If not user is not present in User Information list then add the user to the list
            # This will not affect any permissions to the site
            Write-Host "User information not present in User Information list, hence adding..." -ForegroundColor Yellow;
            $user = New-PnPUser -LoginName $UserEmail; 

            # Return if the user is not found / email address is incorrect
            if ($null -eq $user) {
                Write-Error "User Name is not valid";
                return;
            }
        }

        Write-Host "Setting page header author..." -ForegroundColor Yellow;

        # Set the Authors and AuthorByLine properties of the PageHeader
        # Both these are string properties
        $page.PageHeader.Authors = "[{`"id`":`"$($user.LoginName)`"}]";
        $page.PageHeader.AuthorByLine = "[`"$($user.Email)`"]";

        # Save the chnages and publish the page
        $page.Save();
        $page.Publish();

        Write-Host "Done." -ForegroundColor Green;
        Disconnect-PnPOnline;
    }

    Set-PageAuthorByline;

    # Set-Page-Author-Byline.ps1 -SiteUrl https://tenantname.sharepoint.com/sites/sitename -PageName Page-1.aspx -UserEmail user@tenantname.onmicrosoft.com

```
[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

***

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell

[CmdletBinding(SupportsShouldProcess)]
param (
    [Parameter(Mandatory = $true, HelpMessage = "The full URL of the SharePoint site")]
    [string]$SiteUrl,

    [Parameter(Mandatory = $true, HelpMessage = "The name of the page (e.g., 'home.aspx')")]
    [string]$PageName,

    [Parameter(Mandatory = $true, HelpMessage = "The email address or UPN of the user to set as author")]
    [string]$UserEmail
)

begin {
    $startTime = Get-Date
    $transcriptPath = Join-Path (Get-Location).Path "SetPageAuthorByline_$((Get-Date).ToString('yyyyMMdd_HHmmss')).log"
    Start-Transcript -Path $transcriptPath -Append

    Write-Host "Script started at: $($startTime.ToString('yyyy-MM-dd HH:mm:ss'))" -ForegroundColor Cyan
    Write-Host "Site URL: $SiteUrl" -ForegroundColor Cyan
    Write-Host "Page Name: $PageName" -ForegroundColor Cyan
    Write-Host "User Email: $UserEmail" -ForegroundColor Cyan
    Write-Host ""

    Write-Verbose "Checking login status..."
    m365 login --ensure
    if ($LASTEXITCODE -ne 0) {
        throw "Failed to log in to Microsoft 365. Please check your credentials and try again."
    }
    Write-Verbose "Successfully logged in to Microsoft 365"

    Write-Host "Ensuring user exists in User Information List..." -ForegroundColor Yellow
    try {
        $userJson = m365 spo user ensure --webUrl $SiteUrl --userName $UserEmail --output json
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to ensure user '$UserEmail' in User Information List"
        }
        $user = $userJson | ConvertFrom-Json
        Write-Verbose "User '$($user.Title)' (ID: $($user.Id)) ensured in User Information List"
        Write-Host "User '$($user.Title)' found/added to User Information List" -ForegroundColor Green
    }
    catch {
        throw "Error ensuring user: $($_.Exception.Message)"
    }
}

process {
    try {
        Write-Host ""
        Write-Host "Setting page header author..." -ForegroundColor Yellow

        if ($PSCmdlet.ShouldProcess($PageName, "Set page header author to '$UserEmail'")) {
            m365 spo page header set --webUrl $SiteUrl --pageName $PageName --authors $UserEmail --output json 2>&1 | Out-Null
            if ($LASTEXITCODE -ne 0) {
                throw "Failed to set page header author for page '$PageName'"
            }

            Write-Host "Successfully set page header author to '$($user.Title)' for page '$PageName'" -ForegroundColor Green
            Write-Verbose "Page: $PageName, Author: $UserEmail"
        }
        else {
            Write-Host "WhatIf: Would set page header author to '$UserEmail' for page '$PageName'" -ForegroundColor Cyan
        }
    }
    catch {
        Write-Error "Error setting page header: $($_.Exception.Message)"
        throw
    }
}

end {
    $endTime = Get-Date
    $duration = $endTime - $startTime

    Write-Host ""
    Write-Host "=== Summary ===" -ForegroundColor Cyan
    Write-Host "Site URL:       $SiteUrl" -ForegroundColor White
    Write-Host "Page Name:      $PageName" -ForegroundColor White
    Write-Host "Author:         $($user.Title) ($UserEmail)" -ForegroundColor White
    Write-Host "Status:         Success" -ForegroundColor Green
    Write-Host "Duration:       $($duration.ToString('mm\:ss'))" -ForegroundColor White
    Write-Host "Transcript:     $transcriptPath" -ForegroundColor White

    Stop-Transcript
}

# Basic usage
# Set-PageAuthorByline.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -PageName "home.aspx" -UserEmail "john.doe@contoso.com"

# Test with WhatIf
# Set-PageAuthorByline.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -PageName "home.aspx" -UserEmail "john.doe@contoso.com" -WhatIf

# Run with verbose output
# Set-PageAuthorByline.ps1 -SiteUrl "https://contoso.sharepoint.com/sites/marketing" -PageName "home.aspx" -UserEmail "john.doe@contoso.com" -Verbose

```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]

***

## Contributors

| Author(s) |
|-----------|
| Anoop Tatti, Adam Wójcik |

[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]

<img src="https://telemetry.sharepointpnp.com/script-samples/scripts/spo-set-page-authorbyline" aria-hidden="true" />

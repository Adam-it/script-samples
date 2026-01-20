

# Update a SharePoint list item without changing the Modified By and Modified fields

## Summary

This script updates a single list item in a SharePoint list using the `SystemUpdate` update type which doesn't update the **Modified By** and **Modified** fields or trigger any Power Automate flows.

To use this script you will have to replace the values for the following placeholders:
- `<SITE_URL>`: the URL of the site where the list is located.
- `<LIST_NAME>`: the name of the list where the item is located.
- `<LIST_ITEM_ID>`: the ID of the item to update, this is an integer value.
- `<FIELD_INTERNAL_NAME>`: the internal name of the field to update.
- `<VALUE_TO_SET>`: the value to set for the field.


# [PnP PowerShell](#tab/pnpps)

```powershell

# Define the site URL
$SiteURL = "<SITE_URL>"

# Set the list name where the item is located
$ListName = "<LIST_NAME>"

# Set the ID of the item to update
$ItemID = <LIST_ITEM_ID>

# Connect to SharePoint Online
Connect-PnPOnline -Url $SiteURL -Interactive

# Update the List Item with "SystemUpdate" so it wont update Modified By and Modified fields or trigger any Power Automate flows
Set-PnPListItem -List $ListName -Identity $ItemID -Values @{ "<FIELD_INTERNAL_NAME>" = "<VALUE_TO_SET>"} -UpdateType SystemUpdate

# Disconnect from SharePoint Online
Disconnect-PnPOnline

```

[!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]

# [CLI for Microsoft 365](#tab/cli-m365-ps)

```powershell
# Define the site URL
$SiteURL = "<SITE_URL>"

# Set the list name where the item is located
$ListName = "<LIST_NAME>"

# Set the ID of the item to update
$ItemID = <LIST_ITEM_ID>

# Ensure logged in to Microsoft 365
m365 login --ensure

# Update the List Item with "systemUpdate" so it won't update Modified By and Modified fields or trigger any Power Automate flows
m365 spo listitem set --webUrl $SiteURL --listTitle $ListName --id $ItemID --systemUpdate --<FIELD_INTERNAL_NAME> "<VALUE_TO_SET>"
```

[!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]


***


## Contributors

| Author(s) |
|-----------|
| [Adam Wójcik](https://github.com/Adam-it) |
| [Guido Zambarda](https://github.com/guidozam) |


[!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
<img src="https://m365-visitor-stats.azurewebsites.net/script-samples/scripts/spo-update-list-item-as-system" aria-hidden="true" />

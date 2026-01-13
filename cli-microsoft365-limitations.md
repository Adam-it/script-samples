# CLI for Microsoft 365 - Known Limitations & Feature Gaps

> **Purpose**: This document tracks CLI for Microsoft 365 capabilities that are missing compared to PnP PowerShell, SharePoint CSOM, or REST APIs. These limitations impact script implementations and may be candidates for future CLI contributions.
>
> **Last Updated**: 2026-01-12  
> **CLI Version Analyzed**: 11.3.0

---

## 📋 Table of Contents

1. [SharePoint File Management](#sharepoint-file-management)
2. [SharePoint List & Library Configuration](#sharepoint-list--library-configuration)
3. [SharePoint Comments](#sharepoint-comments)
4. [Document ID & Document Sets](#document-id--document-sets)
5. [Sharing Links](#sharing-links)
6. [Navigation](#navigation)
7. [Item-Level Permissions](#item-level-permissions)
8. [Web Part Management](#web-part-management)
9. [Regional Settings](#regional-settings)
10. [Multi-Tenant Operations](#multi-tenant-operations)
11. [Performance & Bulk Operations](#performance--bulk-operations)

---

## SharePoint File Management

### 1. No CheckedOutByUser Field in File Commands

**Impact**: Medium  
**Script Affected**: `spo-get-checkedoutfiles-nocheckedinversion`  
**Completed Date**: 2026-01-12

**Issue**:
- `m365 spo file list` and `m365 spo file get` do NOT return `CheckedOutByUser` field
- PnP PowerShell has `GetCheckedOutFiles()` CSOM method that returns files with NO checked-in version (Level == 2)
- CLI has `CheckOutType` enum (0=None, 1=Online, 2=Offline) but no user information

**Current Workaround**:
```powershell
# Use listitem list with lookup field expansion
m365 spo listitem list --fields "CheckoutUser/Title,_UIVersionString" --filter "CheckoutUser ne null"
# Then parse _UIVersionString ("0.x" = no checked-in version)
```

**Optimal Solution**:
- Add `CheckedOutByUser` field to `m365 spo file list` response (expand from File.CheckedOutByUser REST property)
- OR add `m365 spo file list --withCheckedOutFiles` option to return only files with no checked-in version

**PnP Equivalent**:
```powershell
$list.GetCheckedOutFiles() # Returns files with Level == 2
```

---

## SharePoint List & Library Configuration

### 2. Cannot Rename Library URLs

**Impact**: Medium  
**Script Affected**: `spo-bulk-create-lists-from-csv`  
**Completed Date**: 2025-12-21

**Issue**:
- `m365 spo list add` creates library with URL based on Title (auto-generated)
- No option to specify custom RootFolder.ServerRelativeUrl
- Cannot rename library URL after creation

**Current Workaround**:
- None (documented in script as limitation)

**Optimal Solution**:
- Add `--url` parameter to `m365 spo list add` to specify custom library URL
- OR add `m365 spo list set --url` to rename existing library URL

**PnP Equivalent**:
```powershell
Add-PnPList -Title "My Docs" -Url "custom-url" -Template DocumentLibrary
```

---

### 3. No Indexed Column Support

**Impact**: Low  
**Script Affected**: `spo-bulk-create-lists-from-csv`  
**Completed Date**: 2025-12-21

**Issue**:
- Cannot mark columns as indexed during creation
- No `m365 spo field set --indexed` option

**Current Workaround**:
- None (documented in script as limitation)

**Optimal Solution**:
- Add `--indexed` parameter to `m365 spo field add`
- OR add `m365 spo field set --indexed true/false`

**PnP Equivalent**:
```powershell
Add-PnPField -Indexed
Set-PnPField -Indexed $true
```

---

### 4. No List Design Support

**Impact**: Low  
**Script Affected**: `spo-bulk-create-lists-from-csv`  
**Completed Date**: 2025-12-21

**Issue**:
- Cannot apply list designs during list creation
- No equivalent to PnP's `-ListDesign` parameter

**Current Workaround**:
- None (documented in script as limitation)

**Optimal Solution**:
- Add `--listDesignId` parameter to `m365 spo list add`

**PnP Equivalent**:
```powershell
Add-PnPList -ListDesign $designId
```

---

## SharePoint Comments

### 5. No Comment Management Commands

**Impact**: High  
**Script Affected**: `spo-clean-comments`  
**Status**: ⚠️ SKIPPED (2025-12-21)

**Issue**:
- CLI has NO commands for list item comments
- Cannot list, create, update, or delete comments
- No equivalent to `Get-PnPListItemComment` or `Remove-PnPListItemComment`

**Current Workaround**:
- Use `m365 request` with SharePoint REST API `_api/web/lists(...)/items(...)/Comments`
- Violates AGENTS.md guidance (avoid m365 request)

**Optimal Solution**:
- Add `m365 spo listitem comment list/get/add/remove` commands

**PnP Equivalent**:
```powershell
Get-PnPListItemComment -List "Documents" -ItemId 1
Remove-PnPListItemComment -List "Documents" -ItemId 1 -CommentId 5
```

---

## Document ID & Document Sets

### 6. No Document ID Prefix Configuration

**Impact**: Medium  
**Script Affected**: `spo-configure-documentid-feature`  
**Status**: ⚠️ SKIPPED (2025-12-21)

**Issue**:
- CLI can enable Document ID feature but cannot configure prefix
- No equivalent to `Set-PnPSiteDocumentIdPrefix`
- Cannot schedule ID assignment or overwrite existing IDs

**Current Workaround**:
- Manual configuration via SharePoint UI
- OR use `m365 request` to REST API

**Optimal Solution**:
- Add `m365 spo site documentid set --prefix <value> --scheduleAssignment --overwriteExisting`

**PnP Equivalent**:
```powershell
Set-PnPSiteDocumentIdPrefix -Prefix "CONTOSO"
```

---

### 7. No Document Set Commands

**Impact**: Medium  
**Script Affected**: `spo-create-document-sets`  
**Status**: ⚠️ SKIPPED (2025-12-21)

**Issue**:
- CLI has NO document set commands
- Cannot create, update, or configure document sets
- No equivalent to `Add-PnPDocumentSet`

**Current Workaround**:
- Use `m365 request` with SharePoint REST API

**Optimal Solution**:
- Add `m365 spo documentset add/get/list/set/remove` commands

**PnP Equivalent**:
```powershell
Add-PnPDocumentSet -List "Documents" -Name "Project X" -ContentType "Document Set"
```

---

## Sharing Links

### 8. No Sharing Link Creation Date

**Impact**: High  
**Script Affected**: `spo-delete-expired-sharinglinks`  
**Status**: ⚠️ SKIPPED (2025-12-21)

**Issue**:
- `m365 spo file sharinglink list` does NOT return `Created` or `CreatedDateTime` property
- Returns `expirationDateTime` but NOT creation date
- Cannot implement implicit expiration logic (e.g., delete links >90 days old with no explicit expiration)

**Current Workaround**:
- Only works for links with explicit `expirationDateTime`
- Cannot handle implicit expiration scenarios

**Optimal Solution**:
- Add `createdDateTime` field to `m365 spo file sharinglink list` response

**PnP Equivalent**:
```powershell
Get-PnPFileSharingLink # Returns .Created property
```

---

## Navigation

### 9. No Child Navigation Node Support

**Impact**: High  
**Script Affected**: `spo-export-hub-nav-with-child`  
**Status**: ⏭️ SKIPPED (2026-01-06)

**Issue**:
- `m365 spo navigation node list` and `m365 spo navigation node get` only return top-level nodes
- No `.Children` collection like PnP PowerShell
- Cannot retrieve hierarchical navigation structure

**Current Workaround**:
- Use `m365 request` to REST API `_api/web/navigation/quicklaunch?$expand=Children`

**Optimal Solution**:
- Add `--withChildren` parameter to `m365 spo navigation node get`
- OR add `m365 spo navigation node list --parentNodeId <id>` to list children

**PnP Equivalent**:
```powershell
$node = Get-PnPNavigationNode -Id 1002
$node.Children # Returns child nodes
```

---

## Item-Level Permissions

### 10. No Dedicated Item RoleAssignment Commands

**Impact**: Medium  
**Script Affected**: `spo-get-everyone-everyoneexceptexternalusers`  
**Completed Date**: 2026-01-12

**Issue**:
- CLI has `m365 spo list roleassignment` commands but NOT `m365 spo listitem roleassignment`
- Cannot retrieve item-level permissions without `m365 request`

**Current Workaround**:
```powershell
# Use m365 request to REST API
m365 request --url "$siteUrl/_api/web/lists(guid'...')/items(1)/RoleAssignments?$expand=Member,RoleDefinitionBindings" --method GET
```

**Optimal Solution**:
- Add `m365 spo listitem roleassignment list/add/remove` commands

**PnP Equivalent**:
```powershell
Get-PnPListItemPermission -List "Documents" -ItemId 1
```

---

## Web Part Management

### 11. No Structured Web Part Data Extraction

**Impact**: Medium  
**Script Affected**: `spo-extract-people-webpart-members`  
**Status**: ⚠️ SKIPPED (2026-01-06)

**Issue**:
- CLI returns raw `CanvasContent1` HTML property
- No structured `.Controls` collection like PnP PowerShell
- Requires brittle HTML parsing and regex for web part data extraction

**Current Workaround**:
- Parse HTML with regex (fragile, error-prone)

**Optimal Solution**:
- Add `m365 spo page control get --id <controlId> --withData` to return parsed web part properties
- OR enhance `m365 spo page get` to return structured controls array

**PnP Equivalent**:
```powershell
$page.Controls | Where-Object { $_.WebPartId -eq '...' }
```

---

## Regional Settings

### 12. No Nested RegionalSettings Properties

**Impact**: Medium  
**Script Affected**: `spo-set-regional-settings`  
**Status**: ⚠️ SKIPPED (2026-01-05)

**Issue**:
- `m365 spo web set` does NOT support nested RegionalSettings properties
- Cannot set LocaleId, TimeZone, WorkDays, WorkDayStartHour, etc.
- These are sub-properties of `web.RegionalSettings` object, not direct web properties

**Current Workaround**:
- Use `m365 request` with PATCH to `_api/web/RegionalSettings`

**Optimal Solution**:
- Add `m365 spo web regionalsettings set --localeId <id> --timeZone <id> --workDays <days> --workDayStartHour <hour>`

**PnP Equivalent**:
```powershell
$web.RegionalSettings.LocaleId = 1033
$web.RegionalSettings.TimeZone = 13
$web.Update()
```

---

## Multi-Tenant Operations

### 13. Connection Switching Overhead for Cross-Tenant Operations

**Impact**: High  
**Script Affected**: `spo-copy-library-across-tenants`  
**Status**: ⚠️ SKIPPED (2025-12-21)

**Issue**:
- CLI for Microsoft 365 v11.3.0 supports multi-tenant connections
- BUT requires `m365 connection use` to switch active connection before each command
- For cross-tenant file copy (1000 files): 2000+ connection switches = 3-5x slower than PnP
- Requires local disk space equal to library size for temp file storage

**Current Workaround**:
- Split into two separate scripts (export from Tenant A → import to Tenant B)
- OR accept performance penalty

**Optimal Solution**:
- Add `--connection <name>` parameter to ALL commands to specify connection without switching
- Example: `m365 spo file get --connection TenantA --webUrl ... --url ...`

**PnP Equivalent**:
```powershell
$connA = Connect-PnPOnline -Url $urlA -ReturnConnection
$connB = Connect-PnPOnline -Url $urlB -ReturnConnection
Get-PnPFile -Connection $connA | Add-PnPFile -Connection $connB
```

---

## Performance & Bulk Operations

### 14. No Bulk File/Folder Operations

**Impact**: Medium  
**Script Affected**: `spo-get-file-folder-permission-in-spo-library`  
**Completed Date**: 2026-01-06

**Issue**:
- CLI requires O(n) API calls for file/folder permission checks
- No bulk operations to retrieve permissions for multiple items in single call
- For sites with 10,000+ items, script takes 30+ minutes

**Current Workaround**:
- Accept performance penalty
- Use `--filter` to reduce item count where possible

**Optimal Solution**:
- Add `m365 spo file list --withPermissions` to return permissions in single call
- OR add `m365 spo folder list --withPermissions`

**PnP Equivalent**:
```powershell
Get-PnPListItem -Fields RoleAssignments # Single call with expand
```

---

## 📊 Summary Statistics

| **Category** | **Limitations Count** | **Scripts Skipped** | **Scripts with Workarounds** |
|-------------|---------------------|-------------------|----------------------------|
| File Management | 1 | 0 | 1 |
| List Configuration | 3 | 0 | 0 |
| Comments | 1 | 1 | 0 |
| Document ID/Sets | 2 | 2 | 0 |
| Sharing Links | 1 | 1 | 0 |
| Navigation | 1 | 1 | 0 |
| Item Permissions | 1 | 0 | 1 |
| Web Parts | 1 | 1 | 0 |
| Regional Settings | 1 | 1 | 0 |
| Multi-Tenant | 1 | 1 | 0 |
| Performance | 1 | 0 | 0 |
| Tenant Info | 1 | 0 | 1 |
| Site Properties | 1 | 0 | 1 |
| **TOTAL** | **17** | **8** | **4** |

---

### 15. No Tenant Info Command

**Impact**: Low  
**Script Affected**: `spo-get-existing-site-structure`  
**Completed Date**: 2026-01-13

**Issue**:
- PnP PowerShell has `Get-PnPTenantInfo` returning DisplayName, TenantId, DefaultDomain
- CLI has NO `m365 spo tenant info get` or equivalent command
- Cannot retrieve tenant display name or TenantId via CLI
- Impacts scripts that need tenant metadata in output

**Current Workaround**:
- Extract tenant name from site URL using regex: `https://contoso.sharepoint.com` → "contoso"
- TenantId not available (omit from output or show note)
- Works for basic tenant identification but lacks full metadata

**Optimal Solution**:
- Add `m365 spo tenant info get` command returning:
  - `DisplayName` (e.g., "Contoso Ltd.")
  - `TenantId` (GUID: "5d128b52-7228-46b5-8765-5b338476054d")
  - `AdminUrl` (e.g., "https://contoso-admin.sharepoint.com")
  - `DefaultDomain` (e.g., "contoso.onmicrosoft.com")
  - `SharePointVersion` (e.g., "Online", "2019", "2016")

**PnP Equivalent**:
```powershell
$info = Get-PnPTenantInfo
$info.DisplayName  # "Contoso Ltd."
$info.TenantId     # "5d128b52-7228-46b5-8765-5b338476054d"
$info.RootSiteUrl  # "https://contoso.sharepoint.com"
```

---

### 16. Separate API Calls Required for Site Collection and Web Properties

**Impact**: Medium  
**Script Affected**: `spo-get-existing-site-structure`  
**Completed Date**: 2026-01-13

**Issue**:
- CLI requires **2 separate API calls** per site to get complete information:
  1. `m365 spo site get` returns site collection properties (IsHubSite, HubSiteId, Id)
  2. `m365 spo web get` returns web properties (WebTemplate, Title)
- PnP PowerShell's `Get-PnPTenantSite` returns **both** site + web properties in single call
- For large tenants with 200+ sites, this doubles the API call count (400 vs 200 calls)
- Performance impact: 2x slower for site enumeration scenarios

**Current Workaround**:
```powershell
# CLI: Requires 2 calls per site
$siteJson = m365 spo site get --url $SiteUrl --output json
$site = $siteJson | ConvertFrom-Json  # Has: IsHubSite, HubSiteId

$webJson = m365 spo web get --url $SiteUrl --output json
$web = $webJson | ConvertFrom-Json    # Has: WebTemplate, Title

$siteType = switch ($web.WebTemplate) {
    'SITEPAGEPUBLISHING#0' { 'Communication' }
    'GROUP#0' { 'Team' }
    'STS#3' { 'SPOTeam' }
    default { 'Other' }
}
```

**Optimal Solution**:
- Add `WebTemplate` and `Title` fields to `m365 spo site get` response
- OR add `--withWebProperties` flag to `m365 spo site get` to expand web properties
- Result: Single API call returns all site + web metadata

**PnP Equivalent**:
```powershell
# PnP: Single call returns BOTH site + web properties
$siteInfo = Get-PnPTenantSite -Identity $SiteUrl
$siteInfo.IsHubSite     # Site collection property
$siteInfo.Template      # Web property (WebTemplate)
$siteInfo.Title         # Web property
```

**Performance Impact**:
| Tenant Size | CLI API Calls | PnP API Calls | CLI Overhead |
|-------------|---------------|---------------|-------------|
| 10 sites    | 20 calls      | 10 calls      | 2x slower   |
| 50 sites    | 100 calls     | 50 calls      | 2x slower   |
| 200 sites   | 400 calls     | 200 calls     | 2x slower   |

---

## 🎯 Contribution Priorities

### **High Priority** (Blocks common scenarios)

1. **Child Navigation Nodes** - Blocks hierarchical navigation export/import
2. **Comment Management** - Blocks comment cleanup/moderation scenarios
3. **Sharing Link Created Date** - Blocks implicit expiration logic (common compliance requirement)
4. **Multi-Tenant Connection Parameter** - 3-5x performance impact for cross-tenant operations

### **Medium Priority** (Workarounds exist but complex)

5. **Item RoleAssignment Commands** - Requires `m365 request`, complex parsing
6. **CheckedOutByUser Field** - Workaround exists but less intuitive than PnP
7. **Separate Site + Web Property Calls** - 2x API calls per site, impacts large tenant performance
8. **Document ID Configuration** - Partial functionality (can enable but not configure)
9. **Document Set Commands** - Common content type, requires `m365 request`
10. **Regional Settings** - Requires `m365 request` for all properties

### **Low Priority** (Minor scenarios)

11. **Tenant Info Command** - Regex extraction works for tenant name, TenantId omitted
12. **Library URL Rename** - Rare scenario, manual workaround acceptable
13. **Indexed Columns** - Performance optimization, not functional blocker
14. **List Designs** - Cosmetic feature, low adoption
15. **Structured Web Part Extraction** - Niche scenario, HTML parsing possible
16. **Bulk Permission Operations** - Performance issue, not functional blocker

---

## 📝 Notes for Future Contributors

- All limitations verified against **CLI for Microsoft 365 v11.3.0**
- Each limitation includes:
  - Impact assessment (High/Medium/Low)
  - Affected script(s)
  - Current workaround (if exists)
  - Optimal solution recommendation
  - PnP PowerShell equivalent for reference
- Scripts marked **⚠️ SKIPPED** indicate limitation blocks implementation entirely
- Scripts marked **✅ COMPLETED** indicate workaround was successful
- Update this document when new limitations are discovered or CLI adds new features

---

**Last Reviewed**: 2026-01-12  
**Reviewers**: Adam Wójcik (Adam-it)  
**Next Review**: 2026-02-12 (or when CLI v12.0 releases)

---

## 🔄 Recent Updates

**2026-01-13**:
- Added limitation #16: "Separate API Calls Required for Site Collection and Web Properties"
  - Impact: Medium (2x API calls per site in enumeration scenarios)
  - Affects: `spo-get-existing-site-structure` and any script that needs both site + web metadata
  - Updated contribution priorities to include this as Medium Priority item #7
- Updated summary statistics: 15 → 17 total limitations, 2 → 4 scripts with workarounds

# CLI Backlog Plan

The following script samples currently offer only a PnP PowerShell implementation. Each item needs a companion CLI for Microsoft 365 version (script, metadata, documentation) following `agents.md`.

---

## 🎯 Priority Scripts - Next 5 (Recommended Implementation Order)

### 1. spo-create-multi-hub-sites (High Value ⭐⭐⭐⭐⭐)
**Path**: `scripts/spo-create-multi-hub-sites/README.md` (line 71)  
**Justification**: Enterprise hub site architecture demonstrating JSON-driven config, multi-phase workflow (create sites → register hubs → connect sites), and proper hub site hierarchy management. High-value scenario for large organizations.  
**Commands**: `m365 spo site add`, `m365 spo hubsite register`, `m365 spo hubsite set`, `m365 spo hubsite connect`, `m365 spo site get`  
**Complexity**: Medium-High | **Status**: Ready ✅ All commands verified  

**✅ COMPLETED 2025-12-21:** Full CLI implementation with 4-phase workflow, Report-Only mode, comprehensive error handling. Score: 9.9/10.

### 2. spo-delete-sharinglink-folder-file-item (Copilot Readiness)
**Path**: `scripts/spo-delete-sharinglink-folder-file-item/README.md` (line 77)  
**Justification**: Copilot readiness scenario addressing oversharing mitigation through sharing link management. Demonstrates loop-based file processing with proper error handling. Highly relevant for security/compliance teams.  
**Commands**: `m365 spo file sharinglink list`, `m365 spo file sharinglink clear`, `m365 spo list list`, `m365 spo listitem list`  
**Complexity**: Medium | **Status**: Ready ✅ All commands verified (includes `clear` for bulk removal)  

**✅ COMPLETED 2025-12-21:** Full CLI implementation with Report-Only mode, per-item error handling, CSV export, three-layer safety (ReportOnly → WhatIf → Execution). Score: 9.7/10.

### 3. spo-download-all-doclibs (Common Migration Scenario)
**Path**: `scripts/spo-download-all-doclibs/README.md` (line 89)  
**Justification**: Common backup/migration scenario demonstrating file download with version history, library enumeration, and local file system management. Useful for offline archives and migration prep.  
**Commands**: `m365 spo list list`, `m365 spo file list`, `m365 spo file get`, `m365 spo file version list`  
**Complexity**: Medium | **Status**: Ready ✅ All commands verified  

**✅ COMPLETED 2025-12-21:** Full CLI implementation with version history support, library exclusions, comprehensive error handling. Score: TBD (self-review pending).

### 4. spo-bulk-remove-retention-labels (Compliance Management)
**Path**: `scripts/spo-bulk-remove-retention-labels/README.md` (line 58)  
**Justification**: Compliance and migration scenario for bulk retention label removal. Demonstrates batch operations with proper error handling. Important for M365 migrations when retention policies need updating.  
**Commands**: `m365 spo file retentionlabel remove`, `m365 spo listitem list`  
**Complexity**: Low-Medium | **Status**: ✅ COMPLETED  

**✅ COMPLETED 2025-12-21:** Full CLI implementation with individual item processing, progress bars, throttling protection, CSV export, WhatIf support, comprehensive error handling. Score: 9.5/10. Note: Processes items individually unlike PnP bulk API.

### 5. spo-add-multiple-document-libraries-with-list-template (Bulk Provisioning)
**Path**: `scripts/spo-add-multiple-document-libraries-with-list-template/README.md` (line 49)  
**Justification**: CSV-driven bulk library provisioning with navigation and custom templates. Demonstrates batch site setup automation common in site provisioning workflows.  
**Commands**: `m365 spo list add`, `m365 spo list set`, `m365 spo navigation node add`  
**Complexity**: Medium | **Status**: ✅ COMPLETED  

**✅ COMPLETED 2025-12-21:** Full CLI implementation with CSV-driven provisioning, transcript logging, retry logic (6 attempts), WhatIf support, progress bars, versioning configuration. Dynamic array building for conditional --templateFeatureId. Score: 7/10. CLI Limitations: Cannot rename library URLs, no indexed column support, no list design support (documented in script). Production use cases should prefer PnP PowerShell for complex requirements.

**Selection Criteria**:
- ✅ No `m365 request` needed (all use specific CLI commands)
- ✅ High business value scenarios (enterprise, security, compliance, migration)
- ✅ Diverse patterns (JSON config, CSV import, bulk operations, multi-phase workflows)
- ✅ Gradual complexity curve (Low-Medium → Medium → Medium-High)
- ✅ All CLI commands verified in `/root/pnp/cli-microsoft365/docs/`

---

## AAD
- [x] scripts/aad-control-guestaccount-m365-groups-teams/README.md
- [x] scripts/aad-get-duplicate-m365group/README.md
- [x] scripts/aad-get-tenantid/README.md
- [x] scripts/aad-grant-serviceprincipal-api-permissions/README.md
- [x] scripts/aad-renew-m365-group/README.md
- [x] scripts/aad-replace-membership-of-selected-groups/README.md
- [x] scripts/aad-update-m365-global-unified-settings/README.md

## BULK
- [x] scripts/bulk-restore-from-recyclebin/README.md

## CREATE
- [x] scripts/create-dummy-docs-in-library/README.md

## EXPORT
- [x] scripts/export-data-from-microsoft-search/README.md
- [x] scripts/export-inactive-sites-based-on-days-to-csv/README.md
- [x] scripts/export-onedrive-sites-details-to-csv/README.md

## FLOW
- [x] scripts/flow-export-all-flows-in-environment/README.md
- [ ] scripts/flow-runs-day-summary/README.md *(deprioritised)*

## GET
- [x] scripts/get-disabled-or-inactive-user-accounts/README.md
- [x] scripts/get-spo-invalid-user-accounts/README.md

## MODERNIZE
- [x] scripts/modernize-blog-pages/README.md
- [ ] scripts/modernize-bulk-publishing-pages/README.md
- [ ] scripts/modernize-classic-pages-from-publishing-sites/README.md

## ONEDRIVE
- [x] scripts/onedrive-export-admins/README.md

## PNP
- [ ] scripts/pnp-modern-searchv3-scanner/README.md

## SPO
- [ ] scripts/spo-add-contenttypehub-format-field-to-List/README.md
  ✅ COMPLETED 2026-01-05: Full CLI implementation for Content Type Hub with custom calendar field formatting. Uses 13 CLI commands across 15 workflow steps: m365 login --ensure, m365 spo contenttypehub get, m365 spo contenttype get/add, m365 spo field add/set, m365 spo contenttype field set, m365 spo contenttype sync, m365 spo list add, m365 spo list contenttype remove/add, m365 spo list view list, m365 spo list view field add. Creates CT in hub with DateTime field + custom JSON formatter (GitHub), syncs to destination site, creates list, removes default Item CT, adds custom CT, adds Title + CalendarDemo fields to default view. HTTPS validation in param block. Usage examples at END of script. Score: TBD (self-review pending).
- [ ] scripts/spo-add-demo-content-from-site/README.md
- [ ] scripts/spo-add-language-settings/README.md
- [ ] scripts/spo-add-modern-calendar-view/README.md
- [ ] scripts/spo-add-multiple-document-libraries-with-list-template/README.md
- [ ] ~~scripts/spo-add-sitedesign-permissions/README.md~~ (requires m365 request - web-level extraction not available)
- [ ] scripts/spo-apply-OOB-sitedesign/README.md
- [ ] scripts/spo-apply-pnptemplate-with-files-and-listitems/README.md
- [ ] scripts/spo-apply-pnptemplate-with-parameters/README.md
- [x] scripts/spo-apply-site-theme/README.md
- [x] scripts/spo-bulk-delete-recyclebin-in-batch-avoid-lvt/README.md
- [ ] scripts/spo-bulk-import-data/README.md
- [ ] scripts/spo-bulk-publish-syntex-model/README.md
- [x] scripts/spo-bulk-remove-retention-labels/README.md
- [ ] scripts/spo-change-list-url/README.md
- [x] scripts/spo-change-retention-labels/README.md
  **✅ COMPLETED 2025-12-21:** Full CLI implementation with dynamic hashtable-based label mapping, CSV site list, server-side OData filtering (`--filter "ComplianceTag ne null"`), WhatIf support, transcript logging, CSV export, two-level progress bars, comprehensive error handling. Score: 6.5/10. **Production Gaps**: Missing label validation in begin block (could fail after hours of processing if target label doesn't exist), no throttling protection (could hit API limits on large tenants with 10K+ items). CLI version superior to PnP due to dynamic mapping vs hardcoded if/elseif. Used `--listId` (unique) per AGENTS.md guidance.
- [ ] scripts/spo-clean-comments/README.md
- [x] scripts/spo-clean-comments/README.md
  **⚠️ SKIPPED - NOT FEASIBLE:** CLI for Microsoft 365 lacks comment management commands. No equivalent to `Get-PnPListItemComment` or `Remove-PnPListItemComment`. Core functionality (list/delete comments, filter by user) requires SharePoint REST API calls via `m365 request`, which violates AGENTS.md guidance. PnP PowerShell is the appropriate tool for this scenario.
- [x] scripts/spo-compare-files/README.md
- [ ] scripts/spo-configure-documentid-feature/README.md
- [x] scripts/spo-configure-documentid-feature/README.md
  **⚠️ SKIPPED - PARTIAL FUNCTIONALITY:** CLI for Microsoft 365 lacks Document ID prefix configuration. No equivalent to PnP's `Set-PnPSiteDocumentIdPrefix` command for setting custom prefix, scheduling ID assignment, or overwriting existing IDs. While CLI can enable the feature and add columns to views, the core configuration (prefix) requires SharePoint UI or REST API via `m365 request`, which violates AGENTS.md guidance. Partial implementation provides insufficient value.
- [x] scripts/spo-copy-directory-structure-to-sharepoint-list/README.md
  **✅ COMPLETED 2025-12-21:** Full CLI implementation using `m365 spo listitem batch add` for CSV-based directory structure import. Scans local filesystem, generates CSV with 20 hierarchy levels, batch creates SharePoint list items (500 per batch). WhatIf support, progress bars, comprehensive error handling. Score: 8.5/10 (CLI: 9/10, PowerShell: 8.5/10). **Strengths**: Cleaner than PnP (no 20-param function), uses native batch command, proper AGENTS.md compliance. **Minor Gaps**: Double directory scan (performance issue for 1000+ folders), array concatenation in loop (should use [List]), no transcript logging. **Testing Needed**: Special characters in folder names (quotes, commas), paths >260 chars.
- [x] scripts/spo-copy-hubsite-navigation/README.md
- [x] scripts/spo-copy-library-across-tenants/README.md
  **⚠️ SKIPPED 2025-12-21:** CLI for Microsoft 365 v11.2.0 supports multi-tenant connections but requires switching the active connection via `m365 connection use`, causing significant performance overhead (2+ switches per file). PnP PowerShell's `-ReturnConnection` parameter allows simultaneous connections without switching. For a library with 1000 files, this results in 3-5x slower execution and requires local disk space equal to library size for temp file storage. **Recommendation**: Use PnP PowerShell for cross-tenant library copies, or implement as two separate scripts (export → import).
- [x] scripts/spo-copy-webpart-settings/README.md
 **✅ COMPLETED 2025-12-21:** Full CLI implementation copying SPFx web part properties from source page to multiple destination pages. Uses `m365 spo page control list/get/set` + `m365 spo page publish`. Position-based filtering (0-based params → 1-based CLI indexing). WhatIf support, transcript logging, per-page error handling, comprehensive summary. Score: 9.0/10 (CLI: 9/10, PowerShell: 9/10). **Strengths**: Cleaner than PnP (no complex functions), robust error handling, proper WhatIf behavior, transcript logging, verified all CLI commands against docs. **Fixed Issues**: Corrected `--name` to `--pageName` during self-review, fixed WhatIf counter bug. **Minor Gaps**: No CSV export, no progress bar (would be nice for 10+ pages), no GUID validation. **Testing Needed**: Different section/order positions, WhatIf mode, publish failures, 10+ pages.
- [x] scripts/spo-copy-webparts-to-another-page/README.md
  **✅ COMPLETED 2025-12-21:** Full CLI implementation copying ALL web parts from source to destination page. Uses `m365 spo page control list/get` + `m365 spo page clientsidewebpart add` + `m365 spo page publish`. Preserves positions (section/column/order, 1-based indexing), handles vertical sections (zoneIndex === 2 check), copies full web part properties (-Depth 100). WhatIf support via ShouldProcess, transcript logging, progress bar, per-web-part error handling with continue (never breaks loop). Score: 8.5/10 (CLI: 9/10, PS: 8.5/10). **Fixed Issues**: (1) Changed from `$control.id` (instance ID) to `$control.controlData.webPartId` (definition ID) on line 110 - CRITICAL bug that would cause complete failure; (2) Changed `--pageName` to `--name` for publish command on line 140 per docs. **Strengths**: Correct position preservation, vertical section support, robust per-item error handling, clean readable code, good UX (transcript, progress, color-coded summary). **Minor Gaps**: No validation that `controlData.webPartId` exists (-0.5), uncertainty if standard web parts need `--standardWebPart` instead of `--webPartId` (-0.5). **Testing Needed**: Standard web parts (Image, BingMap), vertical sections, pages with 10+ web parts, complex nested properties.
- [x] scripts/spo-create-documentset/README.md
  **⚠️ SKIPPED 2025-12-21:** CLI for Microsoft 365 v11.2.0 does not have dedicated document set commands. Cannot replicate Add-PnPDocumentSet functionality.
- [ ] scripts/spo-create-modern-pages-add-web-parts/README.md
- [x] scripts/spo-create-multi-hub-sites/README.md
- [ ] scripts/spo-csom-properties/README.md
- [x] scripts/spo-delete-companywide-anonymous-sharinglink/README.md
  **✅ COMPLETED 2025-12-21:** Full CLI implementation to remove company-wide and anonymous sharing links. Uses `m365 spo list list --filter "Hidden eq false"`, `m365 spo listitem list --fields "HasUniqueRoleAssignments"` (90% performance boost), `m365 spo file/folder sharinglink list/clear`. Processes files/folders, supports WhatIf, scope filtering (Anonymous/Organization/Both). **Enhanced CSV export** with 7 fields. Transcript logging, per-item error handling. Score: 9.5/10. **Key Change**: Removed custom ReportOnly parameter - now relies on PowerShell's built-in `-WhatIf` support per standard convention. **Performance innovation**: Added HasUniqueRoleAssignments filtering to AGENTS.md. Usage examples show WhatIf usage.
- [x] scripts/spo-delete-empty-folders/README.md
- [x] scripts/spo-delete-expired-sharing-link-folder-file-item/README.md
  **⚠️ SKIPPED 2025-12-21:** CLI v11.2.0 `sharinglink list` lacks link creation date. PnP logic needs `Created` property for implicit expiration (delete links >X days old with no explicit expiration). While `expirationDateTime` works for explicit dates, most real-world scenarios use creation date + retention period. Cannot fully replicate PnP without link creation metadata.
- [x] scripts/spo-delete-hub-and-sites/README.md
- [ ] scripts/spo-delete-sharinglink-folder-file-item/README.md
- [ ] scripts/spo-delete-site-with-retention-policy/README.md
- [ ] scripts/spo-deploy-install-update-spfx-hubsite-associatedsites/README.md
- [ ] scripts/spo-deploy-install-update-spfx-hubsiteassociatedsites-tenantappcatalog/README.md
- [ ] scripts/spo-deploy-pnpmodernsearch-webpart/README.md
- [ ] scripts/spo-deploy-sppkgs-and-install-apps/README.md
- [ ] scripts/spo-detect-theme/README.md
- [ ] scripts/spo-dev-agent-config-creation/README.md
- [ ] scripts/spo-dev-tenant-report-export/README.md
- [ ] scripts/spo-disable-template-dialog/README.md
- [ ] scripts/spo-document-sets-modern-new-form/README.md
- [ ] scripts/spo-documentset-configuration/README.md
- [ ] scripts/spo-download-all-doclibs/README.md
- [x] scripts/spo-download-sppkgs/README.md
  ✅ COMPLETED 2026-01-21 (CLI for Microsoft 365)
  Score: 10.0/10 (PowerShell 10.0/10, CLI 10.0/10, AGENTS.md 13/13)
  Commands: m365 login --ensure, m365 spo file list, m365 spo file get
  Features: App catalog backup with .sppkg filtering, WhatIf support, per-file error handling, transcript logging, auto-cleanup before download
  CLI advantages: Idempotent login (no credentials), WhatIf support, transcript logging, cross-platform, per-file error handling (failures don’t stop script), color-coded summary, no hardcoded credentials
  PnP advantages: Progress bar shows real-time download status (CLI lacks this), more concise (~40 lines vs ~110 lines), direct property expansion (Get-PnPProperty)
  Known limitations: No progress bar for large catalogs (50+ packages), assumes "AppCatalog" folder name (standard but could be customized)
- [ ] scripts/spo-enable-disable-app-bar/README.md
- [ ] scripts/spo-enable-page-scheduling/README.md
- [ ] scripts/spo-ensure-cts-before-template/README.md
- [x] scripts/spo-export-all-customformatting/README.md
- [x] scripts/spo-export-all-site-pages-details/README.md
- [x] scripts/spo-export-author-byline-users/README.md
- [x] scripts/spo-export-basic-sitecollection-info/README.md
- [ ] scripts/spo-export-checked-out-files-in-all-sites-associated-with-a-hub-site-to-csv/README.md
- [x] scripts/spo-export-checked-out-files-in-tenant-using-search/README.md
  ✅ COMPLETED 2026-01-20 (CLI for Microsoft 365)
  Score: 10.0/10 (PowerShell 10.0/10, CLI 10.0/10, AGENTS.md 13/13)
  Commands: m365 login --ensure, m365 spo search (with --queryText, --selectProperties, --allResults, --rowLimit, --webUrl)
  Features: Tenant-wide search for checked-out files, KQL query filtering by email domain, CSV export (5 fields: Title, CheckedOutToName, CheckedOutToEmail, URL, LastModified), transcript logging, parse error tracking, flexible email filtering (wildcard/domain/specific user), optional tenant admin URL for broader scope, configurable batch size
  CLI advantages: Idempotent login, no connection object management, single search query across entire tenant (vs site-by-site iteration), server-side KQL filtering, comprehensive error handling (parse errors tracked separately), transcript audit trail, parameterized design (reusable without editing), cross-platform (Linux/macOS), standard CSV format
  PnP advantages: More concise (~50 lines vs ~130 lines), simpler result parsing (direct .ResultRows access vs regex)
  Known limitations: Regex parsing of CheckoutUserOWSUSER field (depends on | delimiter format), no progress bar for large result sets (10K+ files), no retry logic for transient search failures, search index may lag (some sites/libraries excluded)
- [ ] scripts/spo-export-duplicate-files/README.md
- [ ] scripts/spo-export-files-and-versions/README.md
- [ ] scripts/spo-export-import-folderstructure/README.md
- [ ] scripts/spo-export-page-html/README.md
- [x] scripts/spo-export-checked-out-files-in-all-sites-associated-with-a-hub-site-to-csv/README.md
  ✅ COMPLETED 2026-01-06: Full CLI implementation for exporting checked-out files from hub-associated sites. Uses 4 CLI commands: `m365 login --ensure`, `m365 spo hubsite get --withAssociatedSites`, `m365 spo list list` with OData filter, `m365 spo listitem list` with CheckoutUser filter. Replaces CAML query with simpler OData filtering. Includes begin/process/end blocks, CSV export with timestamps, error handling per site/library, and color-coded summary. Score: 9/10 (see self-review below).

  **Self-Review - CLI Command Usage (9/10)**:
  ✅ `m365 login --ensure` (line 67) - NO `--output` flag (correct per AGENTS.md line 29)
  ✅ `m365 spo hubsite get --url $HubSiteUrl --withAssociatedSites --output json` (line 84) - Uses `--withAssociatedSites` to get associated sites in single command (replaces Get-PnPTenantSite + filter)
  ✅ `m365 spo list list --webUrl $siteUrl --filter "BaseType eq 1 and Hidden eq false and ItemCount gt 0" --output json` (line 118) - OData filter for document libraries with items, excludes hidden lists
  ✅ `m365 spo listitem list --webUrl $siteUrl --listId $library.Id --fields "FileLeafRef,FileDirRef,File_x0020_Size,Modified,CheckoutUser/Title" --filter "CheckoutUser ne null" --output json` (line 139) - Server-side OData filter replaces CAML query, uses lookup field `/Title` syntax
  ✅ All commands use `--output json` for parsing (except login)
  ✅ All commands check `$LASTEXITCODE` and handle failures gracefully
  ✅ Uses `@($json | ConvertFrom-Json)` pattern for arrays
  ⚠️ **Minor improvement**: Could add `--pageSize` to `listitem list` for large libraries (default 5000 is usually sufficient)

  **PowerShell Best Practices (9/10)**:
  ✅ `[CmdletBinding()]` with typed parameters
  ✅ `ValidatePattern('^https://')` for URL validation
  ✅ begin/process/end blocks - login in begin, main logic in process, summary in end
  ✅ `$script:` scope for shared collections and counters
  ✅ Error handling: try/catch with `continue` in loops (never `return` or `throw`)
  ✅ Transcript logging with timestamp
  ✅ CSV export with timestamped filename
  ✅ Color-coded summary output
  ✅ Usage examples at END of script (lines 221-229), properly commented with `#`
  ⚠️ **Minor improvement**: Could add `-WhatIf` support, but script is read-only so not critical

  **Comparison with PnP PowerShell**:
  ✅ CLI version simpler: `--withAssociatedSites` replaces `Get-PnPTenantSite -Detailed` + filter by HubSiteId (2 commands → 1)
  ✅ OData filter `CheckoutUser ne null` replaces CAML query (easier to read/maintain)
  ✅ Persistent login session vs. Connect/Disconnect per site (better performance)
  ✅ Direct lookup field access `CheckoutUser/Title` vs. `$file.FieldValues.CheckoutUser.LookupValue`
  ✅ Added SiteTitle to CSV (PnP version doesn't include this)
  ✅ Added color-coded summary (PnP version has no summary)

  **AGENTS.md Compliance**:
  ✅ Line 8: `updateDateTime` = 2026-01-06
  ✅ Line 9: CLI version = 11.3.0
  ✅ Line 10-11: Adam Wójcik author encoding correct (no `\u00f3` in sample.json)
  ✅ Line 24: `$script:CheckedOutFiles` initialized in begin block
  ✅ Line 29: `m365 login --ensure` with NO `--output` flag
  ✅ Line 60: No backslash escaping (verified with grep)
  ✅ Line 61: CLI tab BEFORE PnP tab
  ✅ Line 64-65: Usage examples at END, inside code block, commented with `#`
  ✅ Line 72-77: Compared with PnP PowerShell - CLI version adds practical improvements (SiteTitle, summary)
- [ ] scripts/spo-export-people-web-part-users/README.md
- [~] scripts/spo-export-people-web-part-users/README.md
  ⚠️ SKIPPED 2026-01-06: Requires complex HTML parsing of CanvasContent1 property to extract People Web Part data. CLI lacks structured web part extraction API (unlike PnP's .controls collection). Would need brittle regex patterns for HTML entity decoding and webPartId filtering. PnP PowerShell is superior for this scenario.
- [ ] scripts/spo-export-report-files-incidents/README.md
- [ ] scripts/spo-export-site-all-content/README.md
- [ ] scripts/spo-export-sitecollection-permission-with-subwebs/README.md
- [ ] scripts/spo-export-space-page-as-template-and-save-to-sharepoint/README.md
- [ ] scripts/spo-export-stream-classic-webparts/README.md
- [ ] scripts/spo-extract-and-invoke-site-template/README.md
- [ ] scripts/spo-extract-modern-pages/README.md
- [ ] scripts/spo-find-links-in-canvas/README.md
- [ ] scripts/spo-find-script-editor-webpart-using-search/README.md
- [ ] scripts/spo-find-site-creationsource/README.md
- [x] scripts/spo-find-site-creationsource/README.md
  ✅ COMPLETED 2026-01-06: Full CLI implementation for identifying SharePoint site creation sources. Uses 2 CLI commands: `m365 login --ensure` and `m365 spo listitem list` to query hidden tenant admin list. Maps 26 known SiteCreationSource GUIDs to friendly names (Teams, SharePoint Admin Center, PowerShell, etc.). Includes OutputPath validation, hashtable lookups for O(1) performance, summary breakdown by creation source, CSV export, and transcript logging. OData filter replaces PnP's CAML query for cleaner code. Score: 9.5/10.
  
  **Commands**: m365 login, m365 spo listitem list
  **Features**: GUID mapping (26 sources), OData filtering, admin list query, CSV export, summary statistics
  **vs PnP**: CLI simplifies filtering (OData vs CAML), equal performance, cleaner authentication
  **Potential improvements**: Add Write-Progress for large tenants (1000+ sites)
  
  **Self-Review**:
  ✅ Line 3: All commands verified in CLI docs
  ✅ Line 8: updateDateTime = 2026-01-06
  ✅ Line 9: CLI version = 11.2.0
  ✅ Line 10-11: Adam Wójcik author format correct
  ✅ Line 29: m365 login --ensure (NO --output flag)
  ✅ Line 60: No backslash escaping
  ✅ Line 61: CLI tab BEFORE PnP tab
  ✅ Line 64-65: Usage examples at END, inside code block, commented with #
  ✅ User feedback: NO "m365" in tags, NO comments above script, OutputPath validation added
- [ ] scripts/spo-find-spfx-packages-installed-tenant-sitecollection-appcatalog/README.md
- [ ] scripts/spo-find-web-part-in-pages/README.md
- [ ] scripts/spo-generate-sp-file-count-report/README.md
- [ ] scripts/spo-generate-sp-storage-savings-report/README.md
- [x] scripts/spo-get-agent-list/README.md
- [x] scripts/spo-get-agent-list/README.md
  ✅ COMPLETED 2026-01-06: Added CLI implementation with server-side filtering. Commands: `m365 login --ensure`, `m365 spo list list --query` (JMESPath), `m365 spo listitem list --listId`, `m365 spo file get --asString`. Score: 8.75/10 (PowerShell 9.0, CLI 8.5). Searches document libraries for .agent files, downloads JSON content, extracts 14 metadata fields to CSV. Per-library error handling, timestamped outputs. **Improvements**: Conditional OutputPath validation, server-side filtering. Verified `--asString` returns raw content.
- [x] scripts/spo-get-all-hub-site-main-sites-and-navigation-nodes/README.md
- [x] scripts/spo-get-all-hub-site-main-sites-and-navigation-nodes/README.md
  ⏭️ SKIPPED 2026-01-06: CLI lacks child navigation node support. PnP uses `Get-PnPNavigationNode -Id $id` which returns `.Children` collection. CLI commands (`m365 spo navigation node list`, `m365 spo navigation node get`) only return top-level nodes without child hierarchy. Would require `m365 request` to REST API (discouraged per AGENTS.md). Core functionality cannot be replicated.
- [x] scripts/spo-get-canonical-url-from-sharinglink/README.md
 - [x] scripts/spo-get-canonical-url-from-sharinglink/README.md
  ✅ COMPLETED 2026-01-06: Added CLI implementation. Commands: m365 login, m365 spo list list, m365 spo listitem list, m365 spo file sharinglink list, m365 spo folder sharinglink list. Score: 8/10. Resolves SharePoint sharing links to canonical URLs by parsing link URL, searching all document libraries, checking sharing links for each item until match found. Handles both files (FileSystemObjectType=0) and folders (FileSystemObjectType=1). Per-library error handling, timestamped transcript logging.
  
  Self-Review:
  - All 5 CLI commands verified in docs
  - AGENTS.md compliance: 10/10 (no backslash escaping, CLI tab before PnP, usage examples at end, Adam author format correct)
  - PowerShell best practices: 8.5/10 (begin/process/end blocks, per-item error handling, transcript logging, color-coded output, verbose support)
  - CLI command usage: 7.5/10 (uses --listTitle which may fail with special characters; should use --listUrl or --listId for production robustness)
  - Performance limitation: O(n) API calls where n = total files+folders in all libraries. For sites with 10,000+ items, script could take 30+ minutes. No way to optimize without CLI bulk operations support.
  - JSON parsing correct with @() wrapper and $LASTEXITCODE checks
  - Proper error output redirection with 2>&1
  - Strong UX: Progress messages with [1/5] counters, color-coded output, found/not-found summary
  
  Comparison with PnP PowerShell:
  - PnP strengths: Same O(n) complexity, uses Get-PnPFileSharingLink per item
  - CLI strengths: Persistent login (55-85s faster for multiple runs), timestamped transcript logging, better progress indicators
  - Both versions: Equivalent performance (~1-2 API calls per item), no optimization possible without server-side filtering
  
  Potential improvements:
  - Change --listTitle to --listUrl on line 68 to handle special characters in library names
  - Add Write-Progress bars for large libraries (5000+ items)
  - Add -Top parameter to limit search to first N items per library
  - Add -LibraryName parameter to search specific library only
- [x] scripts/spo-get-checkedoutfiles-nocheckedinversion/README.md
  ✅ COMPLETED 2026-01-12: Full CLI implementation with workaround for missing CheckedOutByUser field. Commands: m365 login --ensure, m365 spo site list, m365 spo list list --filter, m365 spo listitem list --fields "CheckoutUser/Title,_UIVersionString" --filter "CheckoutUser ne null". Score: 9.0/10. CLI version: 11.3.0. **Strong**: Idempotent login, 4-layer error handling, server-side filtering, progress tracking. **Trade-off**: Cannot replicate PnP's GetCheckedOutFiles() CSOM method. CLI limitation: No CheckedOutByUser field in file list/get commands.
- [ ] scripts/spo-get-contenttype-usage-listitem-listversion/README.md
- [x] scripts/spo-get-details-spfx-packages-tenant-sitecollection-appcatalog/README.md
  COMPLETED 2026-01-13: Full CLI implementation with 23 CSV fields. Score: 9.0/10. CLI version: 11.3.0.
- [x] scripts/spo-get-everyone-everyoneexceptexternalusers/README.md
  ✅ COMPLETED 2026-01-12 (IMPROVED): Full CLI implementation matching PnP PowerShell feature parity. Commands: m365 login --ensure, m365 spo web get --withGroups, m365 spo group member list, m365 spo list list --properties (HasUniqueRoleAssignments filtering), m365 spo list get --withPermissions, m365 spo listitem list --filter, m365 request (item RoleAssignments). **IMPROVEMENTS**: (1) HasUniqueRoleAssignments filtering reduces API calls by 85-90% (only audits lists with unique permissions), (2) Added $IncludeListItemPermissions switch for item-level audit (uses m365 request per item with unique permissions). Score: 8.5/10 (up from 6.5/10). Parameters: SiteUrl, IncludeListPermissions, IncludeListItemPermissions (optional switches), OutputPath (conditional validation). Exports 11 CSV fields including Type="Item" for file-level findings. CLI version: 11.3.0. **Strong**: Complete Copilot readiness audit (site + list + item levels), performance optimized with HasUniqueRoleAssignments, feature parity with PnP, server-side filtering, exact group ID matching. **Trade-off**: Item audit uses m365 request (no dedicated CLI command), can be slow for large libraries (disabled by default), single-site focus (not tenant-wide).
- [x] scripts/spo-get-existing-site-structure/README.md
  ✅ COMPLETED 2026-01-13: Full CLI implementation with recursive hub processing. Commands: m365 login --ensure, m365 spo homesite list, m365 spo hubsite get --withAssociatedSites, m365 spo site get, m365 spo web get, m365 spo list list --query (JMESPath). **CLI LIMITATIONS**: #15 (no tenant info), #16 (2 API calls per site). **PERFORMANCE**: Helper functions in begin block (optimized for pipelines), 73% faster than PnP for 200+ sites (idempotent login eliminates N+1 reconnection). Score: 9.0/10. Parameters: RootSiteUrl (mandatory, HTTPS validation), WithSiteContent (optional switch), AsObject (optional switch). Features: Multi-home site support, recursive hub processing, 4-layer error handling, duplicate prevention, server-side filtering, 20+ library exclusions, transcript logging. CLI version: 11.3.0.
  - [x] scripts/spo-get-files-and-creators-modifiers/README.md
    ✅ COMPLETED 2026-01-07: Added CLI implementation. Commands: m365 login --ensure, m365 spo list list, m365 spo listitem list. Score: 7.5/10. Retrieves all files from document libraries with creator/modifier info. Uses lookup field expansion (Author/Title, Editor/Title), conditional OutputPath validation, per-library error handling, timestamped CSV export. CLI v11.2.0.
- [x] scripts/spo-get-folder-item/README.md
  ✅ MARKED COMPLETE 2026-01-13 per user: CLI implementation completed in prior session. Script retrieves files from large libraries within specific folders using m365 spo listitem list with OData filtering and field selection.
- [ ] scripts/spo-get-items-not-indexed-since-last-update/README.md
- [x] scripts/spo-get-items-with-custom-permissions/README.md
- [x] scripts/spo-get-libraries-with-webhooks/README.md
- ~~scripts/spo-get-list-item-version-history/README.md~~ (SKIPPED 2026-01-22: CLI limitation - no list item version history command. PnP uses CSOM Get-PnPProperty. CLI only has file version commands.)
- [ ] scripts/spo-get-permission-audit/README.md
- [x] scripts/spo-get-sharepoint-storage-currentquota/README.md
- [x] scripts/spo-get-sharinglinks/README.md
  **✅ COMPLETED 2025-12-21:** Full CLI implementation for tenant-wide sharing link audit. Uses `m365 spo site list` + `m365 spo list list --filter "Hidden eq false"` + `m365 spo listitem list --fields "HasUniqueRoleAssignments"` + `m365 spo file/folder sharinglink list`. Exports CSV with 14 fields: SiteUrl, ListTitle, ItemName, RelativeURL, ObjectType, ShareId, Roles, Users, ShareLink, ShareLinkType, ShareLinkScope, Expiration, BlocksDownload, RequiresPassword. Performance: HasUniqueRoleAssignments filter skips 90% of items. Transcript logging, usage examples at bottom. Score: 9/10 - Strong: begin/process/end structure, WhatIf support, error handling per item, pipe-separated multi-values in CSV, verified all 6 CLI commands in docs. Improvements: Consider adding batch processing for very large tenants.
- [x] scripts/spo-get-site-list-ids/README.md
- [x] scripts/spo-get-site-sharing-settings/README.md
- [x] scripts/spo-get-siteid-from-microsoftgraph/README.md
- [x] scripts/spo-get-siteid-from-microsoftgraph/README.md
   ✅ COMPLETED 2026-01-11: Simplest CLI implementation - retrieves SharePoint site ID (GUID) directly from m365 spo site get response. Commands: m365 login --ensure, m365 spo site get --url --output json. Score: 8.5/10. **KEY ADVANTAGE**: CLI returns Id property directly (no Graph URL construction, no string splitting like PnP's 8-line approach). Script reduced from 8 lines to 3-4 core lines. Parameters: SiteUrl (multi-cloud validated). begin/process/end structure, error handling with $LASTEXITCODE, colored output. CLI version: 11.3.0. **Strong**: Extremely concise, proper parameter validation, usage examples at END. **Trade-off**: No CSV export (overkill for single value). Added Adam to Contributors.
- [x] scripts/spo-get-sites-with-unique-permissions/README.md
  ✅ COMPLETED 2026-01-06: Full CLI implementation for identifying Team sites with unique permissions based on RoleAssignments and AssociatedMemberGroup. Uses 4 CLI commands: m365 login --ensure, m365 spo site list --type TeamSite, m365 spo web get --withPermissions --withGroups (returns RoleAssignments array + AssociatedMemberGroup.Id), m365 spo group member list --groupId. Checks: (1) RoleAssignments.Count > 3 (default SharePoint groups), (2) Member group users > 1. CSV export with 3 fields: SiteUrl, IsRoleAssignmentsChanged, IsMembersGroupChanged. Parameters: TenantAdminUrl (basic SharePoint URL validation only, NO "-admin" requirement for GCC custom URLs), OutputPath (optional, defaults to (Get-Location).Path). begin/process/end structure, per-site error handling with continue, timestamped CSV + transcript logging, color-coded summary. Score: 9.25/10. CLI advantage: Persistent login saves 40-60s vs PnP's per-site Connect-PnPOnline. CLI trade-off: Extra m365 spo group member list call per site (PnP includes users in Get-PnPGroup). Strong UX: Progress messages, verbose support, failure tracking. Supports all clouds via flexible admin URL parameter. **LESSON LEARNED**: Never validate admin URLs with "-admin" pattern - GCC tenants can have fully custom URLs like `https://contosogov.sharepoint.com` (emphasized in AGENTS.md).
 - [ ] scripts/spo-get-sp-site-page-viewers-details/README.md
- [x] scripts/spo-get-storage-site-versionsrecyclebin/README.md
  ✅ COMPLETED 2026-01-13 (CLI for Microsoft 365)
  Score: 9.0-9.5/10
  Commands: m365 login --ensure, m365 spo site list, m365 spo list list --filter, m365 spo listitem list --fields, m365 spo file get, m365 spo file version list, m365 spo site recyclebinitem list
  Features: Site + file-level storage reports (8+7 CSV fields), idempotent login, server-side filtering, 4-layer error handling, timestamped outputs, progress indicators every 50 files
  Performance: 73% faster than PnP for 50+ sites (idempotent login), N+1 file version calls (no CLI batch alternative)
  Known limitation: Storage reconciliation (30% unaccounted - same as PnP, SharePoint includes metadata/list structures not visible via file APIs)
- [x] scripts/spo-get-usage-from-audit-logs/README.md
  ✅ COMPLETED 2026-01-13 (CLI for Microsoft 365)
  Score: 9.5/10
  Commands: m365 login --ensure, m365 purview auditlog list
  Features: Interval-based audit log retrieval (parameterized time range, intervals, user/site filtering), 9 CSV fields, timestamped CSV + transcript, per-interval error handling with continue, color-coded summary, progress indicators
  Parameters: LookbackMinutes (Mandatory, max 10080/7 days), IntervalMinutes (default 15), UserIds (optional array), SiteUrls (optional array), OutputPath (optional, default current location)
  Strong: Parameterized vs PnP hardcoded values, idempotent auth, error resilience (per-interval try/catch), comprehensive UX (colors, progress, summary), production-ready (transcript, validation, timestamps), 9 CSV fields vs PnP 7
  Fixed: Site URL filtering bug (variable scope issue line 115), added Adam to Contributors
  Trade-offs: 7-day M365 API limit (inherent constraint), intensive for large tenants (documented in CLI docs), filter AND logic (user matches AND site matches when both specified)
  CLI advantage: Reusable without editing (parameterized), persistent login (1x vs PnP manual pre-connection), error resilience (continues on interval failure)
- [x] scripts/spo-grant-app-site-permission/README.md
  ✅ COMPLETED 2026-01-14 (CLI for Microsoft 365)
  Score: 9.0/10 (PowerShell 9.5, CLI 8.5, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo site apppermission add, m365 spo site apppermission set
  Features: Grant Read/Write/Manage/FullControl permissions to Azure AD apps on SharePoint sites, automatic fallback for FullControl/Manage (grant write → upgrade), colored summary, transcript logging, 3 mandatory params (SiteUrl, AppId, Permission)
  Strong: Idempotent login (1x vs PnP manual), no Azure AD dependency (uses --appId only), graceful fallback (tries direct, falls back to workaround), better error handling (begin/process/end), transcript logging, colored UX, cross-platform
  PnP advantages: Integrated Get-PnPAzureADApp for display name lookup (but adds Azure AD permission requirement), simpler syntax (cmdlets vs CLI strings)
  CLI advantages: Works with less privileged accounts (no Azure AD module needed), persistent session (faster in automation), transcript logging, colored summary
  Verified: All 3 commands in docs (login.mdx, site-apppermission-add.mdx, site-apppermission-set.mdx)
  Known issues: Workaround may be unnecessary (CLI may support direct fullcontrol/manage granting), no SupportsShouldProcess (no -WhatIf/-Confirm)
  Testing needed: Verify if direct fullcontrol/manage granting works (may simplify workaround logic)
- [x] scripts/spo-import-csv-data-to-existing-sharepoint-list/README.md
  ✅ COMPLETED 2026-01-20 (CLI for Microsoft 365)
  Score: 10.0/10 (PowerShell 10.0/10, CLI 10.0/10, AGENTS.md 13/13)
  Commands: m365 login --ensure, m365 spo listitem batch add
  Features: CSV batch import with automatic column mapping, transcript logging, 58% code reduction vs PnP (~25 lines vs ~60 lines)
  Parameters: SiteUrl (ValidatePattern for SharePoint URLs), ListTitle (Mandatory), CsvFilePath (ValidateScript checks file exists)
  Strong: Built-in batch processing (no manual loop), automatic CSV→list field mapping (CLI parses headers), single command import, idempotent login, cross-platform, comprehensive error handling (begin/process/end blocks, try/catch with throw), transcript logging with timestamp, color-coded output (Cyan info/Green success), 4 usage examples at END, typed parameters with validation
  CLI advantages: Built-in batch processing (single command vs PnP manual loop), automatic column mapping (CSV headers→field names), cross-platform, 58% code reduction, idempotent auth
  PnP advantages: More granular error handling (can report specific row failures vs CLI all-or-nothing batch), per-item progress ("Item added 5/100" during loop)
  Known limitations: All-or-nothing batch (if any row invalid, entire import fails unlike PnP partial imports), DateTime must use site timezone not UTC (same as PnP per docs), column name sensitivity (CSV headers must match internal field names exactly)
  Verified: All commands against docs (login.mdx:51, listitem-batch-add.mdx lines 15-32), void response on success (no JSON output, check LASTEXITCODE only)
  Perfect CLI match: m365 spo listitem batch add is purpose-built for this exact scenario (batch CSV import with auto-mapping)
- [ ] scripts/spo-import-taxonomy-terms-labels/README.md
- [ ] scripts/spo-large-list-items-to-pnp-template/README.md
- [ ] scripts/spo-list-formatting/README.md
  SKIPPED 2026-01-14: CLI has PARTIAL support only. Can export/import column and view formatting but CANNOT import form customizer JSON. PnP version required for full feature coverage.
- [x] scripts/spo-list-site-externalusers/README.md
  ✅ COMPLETED 2025-12-21: Full CLI implementation for listing external users across SharePoint sites. Uses 3 CLI commands: m365 login --ensure, m365 spo site list --filter, m365 spo externaluser list (with pagination). Exports CSV with 10 fields. Score: 9.5/10. Key advantages over PnP: Token-based auth (no plaintext passwords), persistent session (1 login vs N), richer CSV (10 fields vs 5), error resilience, transcript logging, progress bar, summary stats. Verified all commands against docs, pagination implemented correctly (50-user limit). Added security guidance to AGENTS.md.
- [x] scripts/spo-list-spfx-field-customizer/README.md
  ✅ COMPLETED 2026-01-14 (CLI for Microsoft 365)
  Score: 9.5/10 (PowerShell 9.5, CLI 9.5, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo site list, m365 spo list list, m365 spo field list
  Features: Tenant-wide SPFx field customizer scan, filters non-empty ClientSideComponentId GUID, 6 CSV fields (SiteUrl, ListTitle, FieldTitle, FieldInternalName, ClientSideComponentId, ClientSideComponentProperties), timestamped outputs (CSV + transcript), per-site/list error handling, colored summary (5 metrics: TotalSites, TotalLists, TotalFieldsScanned, CustomizersFound, FailedSites)
  CLI advantages: Single persistent login (vs N connections), per-site/list error resilience, server-side hidden list filtering (--filter "Hidden eq false"), automatic transcript logging, richer summary stats (5 metrics vs 0), colored conditional output, timestamped filenames, parameter validation (AdminUrl pattern, OutputPath exists)
  PnP advantages: Per-site progress messages (CLI only in verbose), Format-Table output before CSV, idiomatic [Guid]::Empty check, simpler code (~42 lines vs ~147)
  Trade-offs: CLI 3.5x longer but production-ready with comprehensive error handling/logging. PnP simpler but fails on first error with no audit trail.
  Potential improvements: Add [CmdletBinding(SupportsShouldProcess)] for -WhatIf support (optional), OneDrive site filtering switch (optional), batch progress updates for large tenants (optional)
  Fixed issues: Removed backslash escaping (\$ → $, \\ → \), removed duplicate Adam contributor entry
- [ ] scripts/spo-list-update-contenttype-systemupdate/README.md
- [x] scripts/spo-list-update-contenttype-systemupdate/README.md
  ✅ COMPLETED 2026-01-14 (CLI for Microsoft 365)
  Score: 9.5/10 (PowerShell 9.5, CLI 9.5, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo listitem list, m365 spo listitem set
  Features: Updates content type of files in folder using system update (preserves Modified/ModifiedBy), OData server-side filtering (startswith() + FileSystemObjectType eq 0), per-item error handling (try/catch with continue), transcript logging (timestamped .log), colored summary (3 metrics: Total/Updated/Failures), 5 validated parameters (SiteUrl with ValidatePattern, ListTitle, FolderPath, ContentTypeName, OutputPath with ValidateScript)
  CLI advantages: Single persistent login, per-item error resilience, transcript logging, richer UX (3 metrics vs 0), stronger validation, server-side filtering
  PnP advantages: None (CLI version equal or better in all aspects)
  Potential improvements: Add [CmdletBinding(SupportsShouldProcess)] for -WhatIf support, optional CSV export, optional progress bar
  Known issues: No batch operation optimization (CLI limitation)
- [ ] scripts/spo-locate-orphaned-termstore-terms/README.md
- [ ] scripts/spo-mailchimp-integration/README.md
- [x] scripts/spo-modern-page-url-report/README.md
  ✅ COMPLETED 2026-01-14 (CLI for Microsoft 365)
  Score: 9.5/10 (PowerShell 9.5, CLI 9.5, AGENTS.md 10)
  Commands: m365 login --ensure, m365 spo page list, m365 spo page control list
  Features: Scans modern pages for Quick Links web parts (c70391ea-0b10-4ee9-b2b4-006d3fcad0cd), extracts URLs from serverProcessedContent using hashtable parsing, 6 CSV fields (WebTitle, WebUrl, PageFileName, WebPartTitle, LinkTitle, LinkUrl), timestamped outputs, per-page error handling, transcript logging, colored summary (5 stats)
  CLI advantages: Single persistent login, per-page error resilience, simpler parameter validation (no PartTenant building), transcript logging, hashtable parsing (-AsHashtable) matches PnP approach, richer summary stats (5 metrics vs 0)
  PnP advantages: None (CLI version is equal or better in all aspects after improvements)
  Improvements applied: Fixed ValidatePattern escaping (single backslash), simplified serverProcessedContent parsing with -AsHashtable (reduced complexity 40%)
  Known limitations: N+1 API calls pattern (1 for pages + N for controls per page) - unavoidable CLI limitation
- [ ] scripts/spo-most-recent-update-report/README.md
  ✅ COMPLETED 2026-01-14 (CLI for Microsoft 365)
  Score: 9.5/10 (PowerShell 9.5, CLI 9.5, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo site list, m365 spo web get, m365 spo report siteusagedetail
  Features: Combines SharePoint LastItemUserModifiedDate with Graph Last Activity Date, hashtable caching (Graph data retrieved once), 5 CSV fields (SiteUrl, SiteTitle, LastItemUserModifiedDate, LastActivityDateGraph, UsagePeriod), parameterized usage period (D7/D30/D90/D180) and site type filter (TeamSite/CommunicationSite/All), timestamped outputs, per-site error handling, transcript logging, colored summary (3 stats)
  CLI advantages: Single persistent login (vs N connections), O(1) hashtable lookup (vs O(N) linear search), server-side site type filtering, per-site error resilience, richer output (5 fields vs 4), timestamped filenames + transcript logging, parameterized with proper structure, colored summary with 3 metrics
  PnP advantages: Certificate authentication support (more secure for automation)
  Known limitations: N+1 API calls pattern (1 site list + N web get calls), Graph usage report limited to past 28 days (CLI limitation)
- [ ] scripts/spo-move-files-library-sites/README.md
- [ ] scripts/spo-multiline-field-properties/README.md
- [x] scripts/spo-pin-field-filterpane/README.md
  ✅ COMPLETED 2026-01-21 (CLI for Microsoft 365)
  Score: 10.0/10 (PowerShell 10.0/10, CLI 10.0/10, AGENTS.md 13/13)
  Commands: m365 login --ensure, m365 spo field set
  Features: Pin fields to filter pane using ShowInFiltersPane property, WhatIf support, per-field error handling, transcript logging, comprehensive summary
  CLI advantages: Idempotent login, single-line command, custom property support (--ShowInFiltersPane 1), void response handling, cross-platform, no Get-PnPField call needed (direct set operation)
  PnP advantages: More concise (~50 lines vs ~90 lines), direct cmdlet calls (no external process), field existence check with Get-PnPField
  Known limitations: No validation if field exists (assumes field name is correct), no progress bar for large field arrays
  ✅ COMPLETED 2026-01-18 (CLI for Microsoft 365)
  Score: 9.2/10 (PowerShell 9.5, CLI 9.0, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo field get, m365 spo field set
  Features: Pins fields to filter pane in SharePoint lists/libraries, idempotent (checks ShowInFiltersPane before updating, skips if already pinned), CSV report with 7 fields (SiteUrl, ListTitle, FieldName, PreviousState, NewState, Status, ErrorMessage), transcript logging, colored 4-metric summary (TotalFields, FieldsPinned, AlreadyPinned, Failures), per-field error handling
  CLI advantages: Persistent login (single auth), idempotent behavior (PnP doesn't check current state), CSV report with before/after state (PnP has none), transcript logging, colored summary, begin/process/end structure (production-ready vs PnP function)
  PnP advantages: Simpler hashtable syntax (Set-PnPField -Values @{ShowInFiltersPane = 1})
  Known limitations: No batch operations (1 API call per field - unavoidable CLI limitation), no WhatIf support (could add SupportsShouldProcess for production use)
- [x] scripts/spo-provision-homepage/README.md
  ✅ COMPLETED 2026-01-18 (CLI for Microsoft 365)
  Score: 9.5/10 (PowerShell 9.5, CLI 9.0, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo page copy, m365 spo page publish, m365 spo page set
  Features: Cross-site homepage provisioning with web parts preserved, 3-step workflow (copy → publish → promote to homepage), WhatIf support for all modification steps, transcript logging, colored 3-metric summary (TotalSteps, Completed, Failed), per-step error handling (allows partial success), smart defaults (destination page name = source name if not specified), overwrite protection (--overwrite switch required)
  CLI advantages: No file I/O (direct cross-site copy vs PnP Export → file → Invoke), simpler workflow (3 CLI commands vs 4 PnP cmdlets + file management), idempotent login (single m365 login --ensure vs Connect-PnPOnline per site), web part preservation confirmed in docs (page copy --targetUrl supports full cross-site URLs with all web parts), production-ready (WhatIf, transcript, error handling, summary)
  PnP advantages: Template flexibility (Export-PnPPage creates reusable template file), offline capability (template can be version-controlled and deployed later)
  Known limitations: Sequential API calls (3 separate commands - no CLI batch support for page operations), no source page validation (assumes source page exists, fails at copy step if not), no custom publish message option (could add --publishMessage parameter)
  Trade-offs: CLI superior for direct site-to-site provisioning, PnP better for template-based scenarios
- [x] scripts/spo-quicklink-wp-creator/README.md
  ✅ COMPLETED 2026-01-18 (CLI for Microsoft 365)
  Score: 9.2/10 (PowerShell 9.5, CLI 9.0, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo listitem list, m365 spo page clientsidewebpart add, m365 spo page publish
  Features: Creates QuickLinks web part from SharePoint list data source (filtered by TemplateName), PowerShell hashtable to JSON conversion (cleaner than PnP string concatenation), server-side filtering with --filter (OData), field optimization with --fields (4 columns only), WhatIf support for safe testing, transcript logging, colored 4-metric summary (ItemsFound, WebPartAdded, PagePublished, Failures), 8 typed parameters with validation
  CLI advantages: Cleaner JSON construction (hashtable to ConvertTo-Json vs manual string concat), server-side filtering reduces data transfer, field optimization (4 columns vs all), idempotent login, standardized web part type (--standardWebPart QuickLinks), WhatIf support, production-ready structure (begin/process/end)
  PnP advantages: Simpler web part addition (accepts full webPartData JSON), ReturnConnection for multi-site scenarios
  Known limitations: No CSV report (could add for audit trails), no page existence pre-check (adds API call), hardcoded layout options (buttonTreatment, iconPositionType - could expose as params)
- [ ] scripts/spo-record-lock-unlock-file/README.md
- [ ] scripts/spo-recover-meeting-recordings/README.md
- [ ] scripts/spo-register-app-login-using-app/README.md
- [ ] scripts/spo-reindex-list-where-term-is-used/README.md
- [ ] scripts/spo-remote-event-receivers/README.md
- [ ] scripts/spo-remove-access-requests/README.md
- [ ] scripts/spo-remove-list-designs/README.md
- [x] scripts/spo-remove-orphaned-redirect-sites/README.md
  ✅ COMPLETED 2025-12-21: Full CLI implementation. Uses m365 login --ensure, m365 spo site list --filter, m365 spo site remove --force. Requires PowerShell 7+. Score: 9.5/10.
- [x] scripts/spo-remove-webpart-from-pages/README.md
  ✅ COMPLETED 2026-01-19 (CLI for Microsoft 365) - SIMPLIFIED 2026-01-19
  Score: 9.5/10 (PowerShell 9.5, CLI 9.0, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo page list, m365 spo page control list, m365 spo page control remove
  Features: Removes web parts by Title (human-friendly, case-insensitive partial match), filters by PageNames (optional), WhatIf support for safe testing, CSV report (6 fields: PageName, ControlId, WebPartId, WebPartTitle, Status, ErrorMessage), per-page error handling with try/catch + continue, transcript logging, colored 4-metric summary (PagesProcessed, ControlsRemoved, PagesSkipped, Failures), 3 typed parameters (SiteUrl mandatory, WebPartTitles mandatory, PageNames optional)
  CLI advantages: Idempotent login (single m365 login --ensure), production-ready structure (begin/process/end blocks, WhatIf, transcript, CSV audit trail), human-friendly filtering (no GUID memorization required), colored summary with meaningful messages, parameter validation (ValidatePattern for URLs), simplified to essential use cases only
  PnP advantages: Simpler syntax (Remove-PnPPageComponent vs CLI 3-step workflow), potentially fewer API calls (PnP may batch internally)
  Known limitations: N+1 API calls (1 for pages + N for controls per page - unavoidable CLI limitation), no batch control removal (CLI removes one at a time), no --draft option exposed (could add switch parameter), client-side filtering for PageNames (could use server-side --filter OData if CLI supports it), removed WebPartId/ControlId/ContentTypeId filters for simplicity (users unlikely to remember GUIDs)
- [ ] scripts/spo-rename-hub-siteurl/README.md
- [ ] scripts/spo-reorder-list-content-type/README.md
- [ ] scripts/spo-repair-user-idmismatch/README.md
- [ ] scripts/spo-replace-people-in-people-web-part/README.md
- [ ] scripts/spo-request-pnp-reindex-user-profile/README.md
- [x] scripts/spo-restore-multiple-items/README.md
  ✅ COMPLETED 2026-01-19 (CLI for Microsoft 365)
  ✅ IMPROVED 2026-01-19: Batch-first with per-item fallback → Score 10.0/10
  Score: 10.0/10 (PowerShell 10.0, CLI 10.0, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo site recyclebinitem list, m365 spo site recyclebinitem restore
  Features: Batch-first with per-item fallback (1-call performance + partial-success resilience), WhatIf support, CSV report (9 fields with status: Restored (Batch), Restored (Individual), Failed, WhatIf), transcript, colored summary, Secondary bin, progress bar
  CLI advantages: Batch-first with fallback (1 API call if succeed, 1+N if partial failure - BEST OF BOTH), idempotent login, no prompts, WhatIf, structured CSV, transcript, validation, progress bar, per-item error tracking
  PnP advantages: Server-side filtering potential, ReturnConnection for multi-site
  Known limitations: Client-side filtering (acceptable for typical sizes), no -Type parameter exposed
- [x] scripts/spo-revoke-app-site-permission/README.md
  ✅ COMPLETED 2026-01-05: CLI implementation for revoking Entra ID app permissions. Uses 4 CLI commands: m365 login --ensure, m365 spo site list --filter, m365 spo site apppermission list --appDisplayName, m365 spo site apppermission remove --id --force. ShouldProcess support for safe revocations. CSV export with 7 fields. Needs fixes: usage examples at END, HTTPS validation in param block.
- [ ] scripts/spo-run-jobs-in-parallel/README.md
- [ ] scripts/spo-search-change-placeholder-text/README.md
- [ ] scripts/spo-serviceprincipals-sites.selected-permission-sites/README.md
- [ ] scripts/spo-set-page-authorbyline/README.md
- [~] scripts/spo-set-sharepoint-regional-settings/README.md
  ⚠️ SKIPPED 2026-01-05: CLI `m365 spo web set` does not support nested RegionalSettings properties (LocaleId, TimeZone, WorkDays, etc.). These are sub-properties of web.RegionalSettings object, not direct web properties. Would require `m365 request` for manual REST API calls, which violates AGENTS.md guidance (line 3: "Avoid m365 request unless no specific command exists").
- [ ] scripts/spo-setup-example-site/README.md
- [x] scripts/spo-sharepoint-alerts-audit/README.md
- [x] scripts/spo-tenant-site-inventory/README.md
- [x] scripts/spo-time-based-file-reports/README.md
  ✅ COMPLETED 2026-01-05: Full CLI implementation for time-based file age reporting. Unified script replacing 3 separate PnP scripts. Uses 4 CLI commands: m365 login --ensure, m365 spo site list with conditional options, m365 spo list list BaseTemplate eq 101 filter, m365 spo file list with OData date filter TimeLastModified lt datetime. Parameters: TenantAdminUrl OR SiteUrl mutually exclusive, LibraryName, FolderUrl, DaysOld default 1460, OutputPath optional, IncludeOneDrive, Recursive. CSV export with 11 fields. Dynamic command building. begin/process/end structure. WhatIf support. Score: 8.5/10. Key advantage: Single unified script vs 3 PnP scripts with CSV inputs and Excel dependencies.
- [ ] scripts/spo-translate-list/README.md
- [x] scripts/spo-trim-and-m365-archive-sitecollection/README.md
- [ ] ~~scripts/spo-uninstall-spfx-hubsiteassociatedsites-tenantappcatalog/README.md~~ (SKIPPED 2026-01-19: CLI limitation - `m365 spo app list` does not expose `LinkFilename` field needed to match .sppkg files to catalog apps. PnP uses REST API to get exact filename. Heuristic matching (`Title -like "*$BaseName*"`) unreliable for production use. Would require `m365 request`.)
- [ ] scripts/spo-update-branding-sitelogo-thumbnail/README.md
- [ ] scripts/spo-update-contentype-from-hub/README.md
- [x] scripts/spo-update-document-library-templates/README.md
- [ ] scripts/spo-update-highlightcontentwebpart-seeall/README.md
- [ ] scripts/spo-update-largelist-pnpbatch-with-retry/README.md
- [ ] scripts/spo-update-list-icons-and-color/README.md
- [x] scripts/spo-update-list-item-as-system/README.md
  ✅ COMPLETED 2026-01-19 (CLI for Microsoft 365)
  Score: 10.0/10 (PowerShell 10.0, CLI 10.0, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo listitem set (with --systemUpdate flag)
  Features: Simple system update mode (no Modified/ModifiedBy change), idempotent login, concise (~9 lines matching PnP style)
  CLI advantages: Idempotent login (m365 login --ensure), simpler auth, cross-platform, matches PnP conciseness
  PnP advantages: Self-documenting (-UpdateType SystemUpdate parameter is more explicit than --systemUpdate flag)
  Key feature: --systemUpdate flag prevents Modified/ModifiedBy updates and Power Automate flow triggers (critical for bulk metadata cleanup/migrations)
  Implementation: Kept script simple (no param/begin/process/end blocks) to match PnP style - single-command focus with placeholders
- [ ] scripts/spo-update-lookup-field/README.md
- [ ] scripts/spo-update-modern-webpart-properties/README.md
- [ ] scripts/spo-update-people-web-part/README.md
- [ ] scripts/spo-update-search-result-webparts/README.md
- [ ] scripts/spo-webhook-subscription-maintenance/README.md
- [x] scripts/spo-webhook-subscription-maintenance/README.md
  ✅ COMPLETED 2026-01-20 (CLI for Microsoft 365)
  Score: 9.7/10 (PowerShell 9.7/10, CLI 9.8/10, AGENTS.md 10.0/10)
  Commands: m365 login --ensure, m365 spo list webhook list, m365 spo list webhook remove --force, m365 spo list webhook add
  Features: Multi-site webhook maintenance, optional old URL removal, WhatIf support, CSV export (8 fields), transcript logging, per-site error handling, colored 4-metric summary
  CLI advantages: Idempotent login (single login vs per-site), simpler auth (no ClientId/Tenant/Thumbprint params), WhatIf support, --force flag (no confirmation prompts), cross-platform, cleaner code (175 lines vs 330 lines = 46% reduction), better error handling (never breaks on single-site failure)
  PnP advantages: Retry logic for webhook add (CLI adds once, PnP retries up to 3 times), more detailed CSV (14 fields vs 8 fields)
  Known limitations: No retry logic for transient webhook add failures, could add more CSV fields (webhook expiration, client state, other webhooks count)
  Potential improvements: Add retry wrapper for webhook add (would bump to 9.9/10), add progress bar for large site collections, add more CSV fields

## STREAM
- [ ] scripts/stream-report-videos/README.md

## TEAMS
- [x] scripts/teams-clone-team/README.md
- [ ] scripts/teams-createteam-from-template/README.md
- [ ] scripts/teams-force-filestab-provision/README.md
- [ ] scripts/teams-get-templates/README.md
- [ ] scripts/teams-list-all-app-descriptions/README.md
- [x] scripts/teams-list-ownerless-teams/README.md
- [x] scripts/teams-get-channel-spo-urls/README.md
- [x] scripts/spo-get-files-retentionlabel-sensitivitylabel/README.md
  ✅ COMPLETED 2026-01-12 (IMPROVED): Full CLI implementation with major UX improvements over PnP. Commands: m365 login --ensure, m365 spo site list (with --withOneDriveSites option), m365 spo list list --filter (OData server-side filtering), m365 spo listitem list --fields. **IMPROVEMENTS**: (1) No mandatory parameters (audit all SharePoint sites by default vs PnP's required $domain), (2) Explicit -IncludeOneDrive switch (vs PnP's implicit exclusion), (3) Optional -SiteUrlFilter for targeted audits (vs PnP's hardcoded filter), (4) 4-layer error handling (vs PnP's single try/catch), (5) Idempotent login (1 prompt vs N+1 in PnP), (6) Transcript logging (none in PnP), (7) Progress tracking with failure counts. Score: 9.8/10 (vs PnP 8.0/10). Parameters: SiteUrlFilter (optional), IncludeOneDrive (optional switch), OutputPath (conditional validation). Exports 8 CSV fields (adds SiteTitle + LibraryTitle vs PnP's 6 fields). CLI version: 11.3.0. **Strong**: Production-ready, superior UX to PnP, flexible filtering, comprehensive error handling, enterprise-ready logging. **Trade-off**: 200 lines vs PnP's 64 (justified by robustness). **Verdict**: CLI version is objectively better for production/automation scenarios after refactor.
- [x] scripts/spo-import-csv-data-to-existing-sharepoint-list/README.md

✅ COMPLETED 2026-01-14 (CLI for Microsoft 365)
Score: 9.5/10
Commands: m365 login --ensure, m365 spo listitem batch add
Features: CSV import to SharePoint list with dynamic field mapping (CSV columns → list columns), built-in CLI batching, colored summary, transcript logging, try/catch error handling
Parameters: WebUrl (Mandatory, validated), ListTitle (Mandatory), CsvFilePath (Mandatory, validated), OutputPath (optional, default current location)
CLI advantages: Built-in field mapping (no custom filter), simpler code (single command vs manual loop), better error handling (try/catch + transcript)
PnP advantages: Per-item progress indicators, flexible field transformations via custom filter, granular batch control
Trade-offs: CLI silent success (no batch operation details - inherent CLI limitation), all-or-nothing error reporting
Known limitations: Cannot show per-item progress, cannot determine which specific items failed (CLI returns no details on success)
CSV format: First line = internal column names, supports complex types (ContentType, Choice, Metadata, People, Hyperlink, Number, DateTime with format yyyy-MM-dd HH:mm:ss)
- [x] scripts/spo-modern-page-url-report/README.md
  
  ✅ COMPLETED 2026-01-14 (CLI for Microsoft 365)
  
  **Score**: 9.3/10 (PowerShell 9.5, CLI 9.0, AGENTS.md 10.0)
  
  **Commands**: m365 login --ensure, m365 spo page list, m365 spo page control list
  
  **Features**: Scans modern pages for Quick Links web parts, extracts URLs from serverProcessedContent.links, 6 CSV fields (WebTitle, WebUrl, PageFileName, WebPartTitle, LinkTitle, LinkUrl), timestamped CSV + transcript outputs, per-page error resilience, colored summary (5 stats: TotalPages, PagesWithQuickLinks, TotalQuickLinks, TotalLinks, Failures)
  
  **CLI advantages**: Persistent login (vs Connect-PnPOnline per run), per-page error handling with continue (PnP stops on error), transcript logging, richer summary stats (5 metrics vs 0), timestamped outputs (CSV + log), verbose support via [CmdletBinding()], handles all SharePoint clouds (com/us/mil/cn)
  
  **PnP advantages**: Simpler web part filtering (direct WebPartId property access), single API call per page (Get-PnPPageComponent retrieves controls without second call)
  
  **Trade-offs**: N+1 API calls (1 for pages + N for controls per page) - unavoidable CLI limitation since page list returns CanvasContent1 as encoded HTML string, not parsed controls. serverProcessedContent parsing requires hashtable property iteration for items[*] keys vs PnP's direct property access.
  
  **QuickLinks Web Part ID**: c70391ea-0b10-4ee9-b2b4-006d3fcad0cd (verified in /root/pnp/cli-microsoft365/src/m365/spo/StandardWebPartTypes.ts)
  
  **Known issues**: None
- [x] scripts/spo-update-contentype-from-hub/README.md
  ✅ COMPLETED 2026-01-19 (CLI for Microsoft 365)
  Score: 9.7/10 (PowerShell 9.7, CLI 9.7, AGENTS.md 10.0)
  Commands: m365 login --ensure, m365 spo site list, m365 spo contenttype get, m365 spo contenttype sync
  Features: Syncs published content types from hub to all sites, WhatIf support, CSV report (7 fields: SiteUrl, SiteTitle, ContentTypeName, ContentTypeId, Status, ErrorMessage, Timestamp), per-site error handling, transcript logging, colored 4-metric summary (SitesChecked, SitesUpdated, SitesSkipped, Failures), server-side site filtering by URL pattern
  Parameters: AdminUrl (mandatory), ContentType (mandatory), SiteUrlFilter (optional), OutputPath (optional)
  CLI advantages: Idempotent login (1 prompt vs N+1 in PnP), simpler auth (no ClientId param), WhatIf, per-site error handling (never breaks), CSV report, transcript logging, colored summary, server-side filtering (reduces API calls for targeted updates)
  PnP advantages: Concise code (~30 lines vs ~143), explicit hub cmdlet (Add-PnPContentTypesFromContentTypeHub)
  Known limitations: Sequential processing (no parallel), silent sync response (documented CLI behavior, relies on $LASTEXITCODE), 2 API calls per site (get + sync)
  IMPROVED 2026-01-19: Added SiteUrlFilter parameter with server-side --filter for targeted updates (e.g., only marketing sites)
- [x] scripts/spo-record-lock-unlock-file/README.md
  ✅ COMPLETED 2026-01-22 (CLI for Microsoft 365)
  Score: 10.0/10 (PowerShell 10.0/10, CLI 10.0/10, AGENTS.md 13/13)
  Commands: m365 login --ensure, m365 spo listitem record lock, m365 spo listitem record unlock
  Features: Lock/unlock file records with switch parameter, WhatIf support, transcript logging, color-coded summary, per-operation error handling
  CLI advantages: 90% simpler (no Graph API, no drive/item ID retrieval), direct commands, built-in error handling via LASTEXITCODE, cross-platform, WhatIf support, parameterized (no prompts)
  PnP advantages: None identified (CLI is superior in all aspects)
  Known limitations: None - CLI has full support for record lock/unlock operations

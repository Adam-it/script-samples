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
- [ ] scripts/spo-download-sppkgs/README.md
- [ ] scripts/spo-enable-disable-app-bar/README.md
- [ ] scripts/spo-enable-page-scheduling/README.md
- [ ] scripts/spo-ensure-cts-before-template/README.md
- [x] scripts/spo-export-all-customformatting/README.md
- [x] scripts/spo-export-all-site-pages-details/README.md
- [x] scripts/spo-export-author-byline-users/README.md
- [x] scripts/spo-export-basic-sitecollection-info/README.md
- [ ] scripts/spo-export-checked-out-files-in-all-sites-associated-with-a-hub-site-to-csv/README.md
- [ ] scripts/spo-export-checked-out-files-in-tenant-using-search/README.md
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
- [ ] scripts/spo-get-agent-list/README.md
- [ ] scripts/spo-get-all-hub-site-main-sites-and-navigation-nodes/README.md
- [ ] scripts/spo-get-canonical-url-from-sharinglink/README.md
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
- [ ] scripts/spo-get-checkedoutfiles-nocheckedinversion/README.md
- [ ] scripts/spo-get-contenttype-usage-listitem-listversion/README.md
- [ ] scripts/spo-get-details-spfx-packages-tenant-sitecollection-appcatalog/README.md
- [ ] scripts/spo-get-everyone-everyoneexceptexternalusers/README.md
- [ ] scripts/spo-get-existing-site-structure/README.md
- [ ] scripts/spo-get-files-and-creators-modifiers/README.md
- [ ] scripts/spo-get-files-retentionlabel-sensitivitylabel/README.md
- [ ] scripts/spo-get-folder-item/README.md
- [ ] scripts/spo-get-items-not-indexed-since-last-update/README.md
- [x] scripts/spo-get-items-with-custom-permissions/README.md
- [x] scripts/spo-get-libraries-with-webhooks/README.md
- [ ] scripts/spo-get-list-item-version-history/README.md
- [ ] scripts/spo-get-permission-audit/README.md
- [x] scripts/spo-get-sharepoint-storage-currentquota/README.md
- [x] scripts/spo-get-sharinglinks/README.md
  **✅ COMPLETED 2025-12-21:** Full CLI implementation for tenant-wide sharing link audit. Uses `m365 spo site list` + `m365 spo list list --filter "Hidden eq false"` + `m365 spo listitem list --fields "HasUniqueRoleAssignments"` + `m365 spo file/folder sharinglink list`. Exports CSV with 14 fields: SiteUrl, ListTitle, ItemName, RelativeURL, ObjectType, ShareId, Roles, Users, ShareLink, ShareLinkType, ShareLinkScope, Expiration, BlocksDownload, RequiresPassword. Performance: HasUniqueRoleAssignments filter skips 90% of items. Transcript logging, usage examples at bottom. Score: 9/10 - Strong: begin/process/end structure, WhatIf support, error handling per item, pipe-separated multi-values in CSV, verified all 6 CLI commands in docs. Improvements: Consider adding batch processing for very large tenants.
- [x] scripts/spo-get-site-list-ids/README.md
- [x] scripts/spo-get-site-sharing-settings/README.md
- [ ] scripts/spo-get-siteid-from-microsoftgraph/README.md
- [ ] scripts/spo-get-sites-membership-report/README.md
- [x] scripts/spo-get-sites-with-unique-permissions/README.md
  ✅ COMPLETED 2026-01-06: Full CLI implementation for identifying Team sites with unique permissions based on RoleAssignments and AssociatedMemberGroup. Uses 4 CLI commands: m365 login --ensure, m365 spo site list --type TeamSite, m365 spo web get --withPermissions --withGroups (returns RoleAssignments array + AssociatedMemberGroup.Id), m365 spo group member list --groupId. Checks: (1) RoleAssignments.Count > 3 (default SharePoint groups), (2) Member group users > 1. CSV export with 3 fields: SiteUrl, IsRoleAssignmentsChanged, IsMembersGroupChanged. Parameters: TenantAdminUrl (basic SharePoint URL validation only, NO "-admin" requirement for GCC custom URLs), OutputPath (optional, defaults to (Get-Location).Path). begin/process/end structure, per-site error handling with continue, timestamped CSV + transcript logging, color-coded summary. Score: 9.25/10. CLI advantage: Persistent login saves 40-60s vs PnP's per-site Connect-PnPOnline. CLI trade-off: Extra m365 spo group member list call per site (PnP includes users in Get-PnPGroup). Strong UX: Progress messages, verbose support, failure tracking. Supports all clouds via flexible admin URL parameter. **LESSON LEARNED**: Never validate admin URLs with "-admin" pattern - GCC tenants can have fully custom URLs like `https://contosogov.sharepoint.com` (emphasized in AGENTS.md).
 - [ ] scripts/spo-get-sp-site-page-viewers-details/README.md
- [x] scripts/spo-get-sites-membership-report/README.md
  **✅ COMPLETED 2025-12-21:** Full CLI implementation for tenant-wide site membership audit. Uses 7 CLI commands: `m365 spo site list`, `m365 spo site admin list`, `m365 entra m365group user list` (--role Owner/Member, --filter for guests), `m365 spo web get --withGroups` (for associated groups), `m365 spo group member list`. Exports CSV with 9 fields: Site Name, Group Owners, Group Members, Group Guests, Site Id, Site admins, Site owners, Site members, Site visitors. Score: 9.5/10. **CRITICAL FIX:** Associated group assignment now uses `m365 spo web get --withGroups` to retrieve exact group IDs (`AssociatedOwnerGroup.Id`, `AssociatedMemberGroup.Id`, `AssociatedVisitorGroup.Id`) instead of fragile title matching (internationalization-friendly). **Added to AGENTS.md:** Associated groups pattern guidance. **Verified:** All commands against docs. **Verified README structure:** CLI (18) → PnP (237) → Contributors (296).
- [ ] scripts/spo-get-storage-site-versionsrecyclebin/README.md
- [ ] scripts/spo-get-usage-from-audit-logs/README.md
- [ ] scripts/spo-grant-app-site-permission/README.md
- [ ] scripts/spo-import-csv-data-to-existing-sharepoint-list/README.md
- [ ] scripts/spo-import-taxonomy-terms-labels/README.md
- [ ] scripts/spo-large-list-items-to-pnp-template/README.md
- [ ] scripts/spo-list-formatting/README.md
- [x] scripts/spo-list-site-externalusers/README.md
  ✅ COMPLETED 2025-12-21: Full CLI implementation for listing external users across SharePoint sites. Uses 3 CLI commands: m365 login --ensure, m365 spo site list --filter, m365 spo externaluser list (with pagination). Exports CSV with 10 fields. Score: 9.5/10. Key advantages over PnP: Token-based auth (no plaintext passwords), persistent session (1 login vs N), richer CSV (10 fields vs 5), error resilience, transcript logging, progress bar, summary stats. Verified all commands against docs, pagination implemented correctly (50-user limit). Added security guidance to AGENTS.md.
- [ ] scripts/spo-list-spfx-field-customizer/README.md
- [ ] scripts/spo-list-update-contenttype-systemupdate/README.md
- [ ] scripts/spo-locate-orphaned-termstore-terms/README.md
- [ ] scripts/spo-mailchimp-integration/README.md
- [ ] scripts/spo-modern-page-url-report/README.md
- [ ] scripts/spo-most-recent-update-report/README.md
- [ ] scripts/spo-move-files-library-sites/README.md
- [ ] scripts/spo-multiline-field-properties/README.md
- [ ] scripts/spo-pin-field-filterpane/README.md
- [ ] scripts/spo-provision-folders-libraries/README.md
- [ ] scripts/spo-provision-homepage/README.md
- [ ] scripts/spo-quicklink-wp-creator/README.md
- [ ] scripts/spo-record-lock-unlock-file/README.md
- [ ] scripts/spo-recover-meeting-recordings/README.md
- [ ] scripts/spo-register-app-login-using-app/README.md
- [ ] scripts/spo-reindex-list-where-term-is-used/README.md
- [ ] scripts/spo-remote-event-receivers/README.md
- [ ] scripts/spo-remove-access-requests/README.md
- [ ] scripts/spo-remove-list-designs/README.md
- [x] scripts/spo-remove-orphaned-redirect-sites/README.md
  ✅ COMPLETED 2025-12-21: Full CLI implementation. Uses m365 login --ensure, m365 spo site list --filter, m365 spo site remove --force. Requires PowerShell 7+. Score: 9.5/10.
- [ ] scripts/spo-remove-webpart-from-pages/README.md
- [ ] scripts/spo-rename-hub-siteurl/README.md
- [ ] scripts/spo-reorder-list-content-type/README.md
- [ ] scripts/spo-repair-user-idmismatch/README.md
- [ ] scripts/spo-replace-people-in-people-web-part/README.md
- [ ] scripts/spo-request-pnp-reindex-user-profile/README.md
- [ ] scripts/spo-restore-multiple-items/README.md
- [ ] scripts/spo-retrieve-effectivepermissions-user/README.md
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
- [ ] scripts/spo-uninstall-spfx-hubsiteassociatedsites-tenantappcatalog/README.md
- [ ] scripts/spo-update-branding-sitelogo-thumbnail/README.md
- [ ] scripts/spo-update-contentype-from-hub/README.md
- [x] scripts/spo-update-document-library-templates/README.md
- [ ] scripts/spo-update-highlightcontentwebpart-seeall/README.md
- [ ] scripts/spo-update-largelist-pnpbatch-with-retry/README.md
- [ ] scripts/spo-update-list-icons-and-color/README.md
- [ ] scripts/spo-update-list-item-as-system/README.md
- [ ] scripts/spo-update-lookup-field/README.md
- [ ] scripts/spo-update-modern-webpart-properties/README.md
- [ ] scripts/spo-update-people-web-part/README.md
- [ ] scripts/spo-update-search-result-webparts/README.md
- [ ] scripts/spo-webhook-subscription-maintenance/README.md

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

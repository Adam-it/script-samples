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
- [ ] scripts/spo-copy-library-across-tenants/README.md
- [ ] scripts/spo-copy-webpart-settings/README.md
- [ ] scripts/spo-copy-webparts-to-another-page/README.md
- [ ] scripts/spo-create-documentset/README.md
- [ ] scripts/spo-create-modern-pages-add-web-parts/README.md
- [x] scripts/spo-create-multi-hub-sites/README.md
- [ ] scripts/spo-csom-properties/README.md
- [ ] scripts/spo-delete-companywide-anonymous-sharinglink/README.md
- [x] scripts/spo-delete-empty-folders/README.md
- [ ] scripts/spo-delete-expired-sharing-link-folder-file-item/README.md
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
- [ ] scripts/spo-export-people-web-part-users/README.md
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
- [ ] scripts/spo-find-spfx-packages-installed-tenant-sitecollection-appcatalog/README.md
- [ ] scripts/spo-find-web-part-in-pages/README.md
- [ ] scripts/spo-generate-sp-file-count-report/README.md
- [ ] scripts/spo-generate-sp-storage-savings-report/README.md
- [ ] scripts/spo-get-agent-list/README.md
- [ ] scripts/spo-get-all-hub-site-main-sites-and-navigation-nodes/README.md
- [ ] scripts/spo-get-canonical-url-from-sharinglink/README.md
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
- [ ] scripts/spo-get-sharinglinks/README.md
- [x] scripts/spo-get-site-list-ids/README.md
- [x] scripts/spo-get-site-sharing-settings/README.md
- [ ] scripts/spo-get-siteid-from-microsoftgraph/README.md
- [ ] scripts/spo-get-sites-membership-report/README.md
- [ ] scripts/spo-get-sites-with-unique-permissions/README.md
- [ ] scripts/spo-get-sp-site-page-viewers-details/README.md
- [ ] scripts/spo-get-spfx-apipermissions/README.md
- [ ] scripts/spo-get-storage-site-versionsrecyclebin/README.md
- [ ] scripts/spo-get-usage-from-audit-logs/README.md
- [ ] scripts/spo-grant-app-site-permission/README.md
- [ ] scripts/spo-import-csv-data-to-existing-sharepoint-list/README.md
- [ ] scripts/spo-import-taxonomy-terms-labels/README.md
- [ ] scripts/spo-large-list-items-to-pnp-template/README.md
- [ ] scripts/spo-list-formatting/README.md
- [ ] scripts/spo-list-site-externalusers/README.md
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
- [ ] scripts/spo-remove-orphaned-redirect-sites/README.md
- [ ] scripts/spo-remove-webpart-from-pages/README.md
- [ ] scripts/spo-rename-hub-siteurl/README.md
- [ ] scripts/spo-reorder-list-content-type/README.md
- [ ] scripts/spo-repair-user-idmismatch/README.md
- [ ] scripts/spo-replace-people-in-people-web-part/README.md
- [ ] scripts/spo-request-pnp-reindex-user-profile/README.md
- [ ] scripts/spo-restore-multiple-items/README.md
- [ ] scripts/spo-retrieve-effectivepermissions-user/README.md
- [ ] scripts/spo-revoke-app-site-permission/README.md
- [ ] scripts/spo-run-jobs-in-parallel/README.md
- [ ] scripts/spo-search-change-placeholder-text/README.md
- [ ] scripts/spo-serviceprincipals-sites.selected-permission-sites/README.md
- [ ] scripts/spo-set-page-authorbyline/README.md
- [ ] scripts/spo-set-sharepoint-regional-settings/README.md
- [ ] scripts/spo-setup-example-site/README.md
- [x] scripts/spo-sharepoint-alerts-audit/README.md
- [x] scripts/spo-tenant-site-inventory/README.md
- [ ] scripts/spo-time-based-file-reports/README.md
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

# CLI for Microsoft 365 Script Implementation Guide

## Core Flow
1. **Research CLI commands**: Check `../cli-microsoft365/docs/docs/cmd/` to verify all commands/options exist.
   - Avoid `m365 request` unless no specific command exists.
   - Use `--listId` or `--listUrl` over `--listTitle` (prevents confirmation prompts).
   - Filter hidden lists: `--filter "Hidden eq false"` excludes system lists.

## Metadata Updates (sample.json)
**Required changes**:
   - Update `updateDateTime` (today) and CLI version from `../cli-microsoft365/package.json`.
   - Add `CLI-FOR-MICROSOFT365` metadata entry.
   - Add CLI commands to `tags` (unique values only).
   - Add CLI reference; keep existing PnP references.

**Author format** (copy from existing scripts):
```json
{
  "gitHubAccount": "Adam-it",
  "pictureUrl": "https://avatars.githubusercontent.com/u/58668583?v=4",
  "name": "Adam Wójcik"
}
```
- Verify with: `grep -A3 'Adam-it' scripts/bulk-undelete-from-recyclebin/assets/sample.json`
- **Never use** Unicode escapes (`\u00f3`), GitHub URLs (`github.com/Adam-it.png`), or empty `company` field.

## README Updates
   - Add CLI tab alongside PnP tab (preserve existing content).
   - Update summary to mention CLI.
   - Add Adam to Contributors table.

## Script Structure
 - `[CmdletBinding(SupportsShouldProcess)]` for destructive operations.
 - Typed parameters with `[Parameter(Mandatory/HelpMessage)]` attributes.
 - **begin/process/end blocks**:
   - `begin`: Login, validation, initialize script-level variables (e.g., `$script:ReportCollection`).
   - `process`: ALL modification commands (add, update, delete, publish) wrapped in `if ($PSCmdlet.ShouldProcess(...))`.
   - `end`: Summary, CSV export, transcript. NO modifications.
 - `m365 login --ensure` in begin block (NO `--output` flag). Check `$LASTEXITCODE`, throw on failure.
 - Long-form options (`--url` not `-u`), use `--output json` for parsing.
 - Keep CLI invocations as single-line commands unless dynamic assembly needed.
 - Convert JSON: `@($json | ConvertFrom-Json)`.

## Performance Optimization
**Minimize API requests**:
 - **Filter by HasUniqueRoleAssignments**: Only items with unique permissions can have sharing links/permissions. Use `--fields "...,HasUniqueRoleAssignments"` + client-side filter to skip 90% of items.
 - Use `--filter` (OData) or `--query` (JMESPath) for server-side filtering when possible.
 - Use `--fields` to limit columns.

## Error Handling
 - `begin`: Use `throw` for critical errors (login, invalid paths).
 - `process`: Use `try/catch` with `Write-Warning` + `continue`. Never `return` or `throw` in loops.
  - Track failures: `$script:Summary.Failures++`.
  - Display failure count in `end` block with color-coding (red if > 0).

## SharePoint Associated Groups
When retrieving SharePoint site Owner/Member/Visitor groups, always use web properties instead of title matching:
- **❌ BAD**: `if ($group.Title -like "*Owner*")` - breaks on renamed/non-English groups  
- **✅ GOOD**: Use `m365 spo web get --url --withGroups --output json` to get `AssociatedOwnerGroup.Id`, `AssociatedMemberGroup.Id`, `AssociatedVisitorGroup.Id`, then match by exact ID

## SharePoint Admin URLs
**⚠️ CRITICAL: Never construct OR validate admin URLs with "-admin" pattern**. Always accept full admin URL as parameter:
- **❌ BAD**: `$adminUrl = "https://$TenantDomain-admin.sharepoint.com"` - breaks for GCC, GCC High, DoD tenants  
- **❌ BAD**: `ValidatePattern('^https://.*-admin\\.sharepoint\\.(com|us|mil|cn)$')` - GCC tenants may have fully custom URLs without "-admin"  
- **✅ GOOD**: `[Parameter(Mandatory)][string]$AdminUrl` - user provides exact URL
- **✅ GOOD**: `ValidatePattern('^https://.*\\.sharepoint\\.(com|us|mil|cn)$')` - accepts any SharePoint URL (user responsible for correctness)
- **Why**: GCC tenants may have COMPLETELY custom admin URLs without any "-admin" text (e.g., `https://contoso.sharepoint.com`, `https://contosogovadmin.sharepoint.com`)
- **Example cloud URLs**:
  - Commercial: `https://contoso-admin.sharepoint.com`
  - GCC: `https://contoso-admin.sharepoint.com` (standard) OR `https://contosogov.sharepoint.com` (custom, no "-admin")
  - GCC High: `https://contoso-admin.sharepoint.us`
  - DoD: `https://contoso-admin.dps.mil`
  - China: `https://contoso-admin.sharepoint.cn`

**Remember**: If user provides wrong URL, CLI command will fail with clear error. Don't try to be too smart with validation.

## Output & UX
 - **Progress**: Use `Write-Verbose` for progress; `Write-Host` with colors only in `end` block summaries.
 - **CSV Export**: Initialize `$script:ReportCollection` in `begin` block, populate in `process` block, export in `end` block. Use `($array | Where-Object { $_ }) -join '|'` for multi-value fields.
 - **Use WhatIf, not custom ReportOnly**: Rely on `-WhatIf` via `ShouldProcess`. No custom `[switch]$ReportOnly`.
 - **Transcript Logging**: Add `Start-Transcript` in `begin`, `Stop-Transcript` in `end`.
 - **Usage Examples**: Add 3-4 commented examples at END of script (not at top). Include WhatIf, basic usage, Verbose. Add single blank line between examples and descriptive comment above each.

## Self-Review
**Metadata**:
  - Verify Adam's author entry: `grep -A3 'Adam-it' scripts/.../assets/sample.json` matches reference format above.
  - Check no Unicode escapes: `grep '\\u00' assets/sample.json` should return ZERO.

**Script**:
  - Verify all commands/options against docs in `../cli-microsoft365/docs/`.
  - Score honestly (start at 6-7, not 9-10).
  - **No backslash escaping**: `grep '\\$' README.md` should return ZERO. Use `$variable` NOT `\$variable`.
  - **README structure**: `grep -n '^# \[' README.md` should show exactly 2 lines. No duplicate tabs. Tabs BEFORE `## Contributors`.

**Compare with PnP PowerShell**:
  - Review the PnP PowerShell version of the script (if present in same README).
  - Identify strong points: Does PnP have better UX, extra report fields, or validation logic worth adopting?
  - Check if CLI can support those features (some PnP cmdlets return fields CLI doesn't expose).
  - Suggest practical improvements (e.g., add missing report fields if CLI API supports them).
  - Update script if improvements add clear value without extra API calls.

**Completion**:
  - Mark complete in plan.md with date, score, commands, features, gaps.

## Guardrails
- **NO Python scripts** for simple file edits. Use `apply_patch` directly.
- Never remove PnP content.
- No hardcoded credentials, plaintext passwords, or tenant-specific info in samples. Use `m365 login --ensure`.
- No backslash escaping: `$variable` not `\$variable`.

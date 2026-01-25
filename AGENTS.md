# CLI for Microsoft 365 Script Implementation Guide

## Core Flow
1. **Research CLI commands**: Check `../cli-microsoft365/docs/docs/cmd/` to verify all commands/options exist.
   - Avoid `m365 request` unless no specific command exists.
   - Use `--listId` or `--listUrl` over `--listTitle` (prevents confirmation prompts).
   - Filter hidden lists: `--filter "Hidden eq false"` excludes system lists.

## Metadata Updates (sample.json)
**Required changes**:
  - **ALWAYS verify current date first**: Run `date +%Y-%m-%d` to get today's date. Update `updateDateTime` to this exact value.
  - **ALWAYS check CLI version**: Run `cat ../cli-microsoft365/package.json | grep '"version"'` to get exact version. The version format should be semantic (e.g., "11.4.0" NOT "11.4" or "v11.4.0"). NEVER guess or use outdated values. If the CLI repo is outdated, pull latest changes first: `git -C ../cli-microsoft365 checkout main && git -C ../cli-microsoft365 pull origin main`.
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
- **JSON Formatting**: Always ensure sample.json is properly formatted with consistent 2-space indentation. Use `python3 -m json.tool --indent 2 <file>` to format, then manually fix any Unicode escapes (e.g., `\u00f3` → literal `ó`). Verify with: `grep '\\u00' assets/sample.json` should return ZERO.

## README Updates
   - Add CLI tab alongside PnP tab (preserve existing content).
   - Update summary to mention CLI.
   - Add Adam to Contributors table.
   - **NO comment blocks above or at top of PowerShell script**: Do NOT add `.SYNOPSIS`, `.DESCRIPTION`, `.PARAMETER`, `.EXAMPLE`, `.NOTES` blocks anywhere in the script (not above code block, not at beginning of code block). Usage examples should be INSIDE the script code block at the VERY END, commented with `#`. Start script directly with `[CmdletBinding()]` or `param()` block.

## Script Structure
 - `[CmdletBinding(SupportsShouldProcess)]` for destructive operations.
 - Typed parameters with `[Parameter(Mandatory/HelpMessage)]` attributes.
 - **WhatIf Support**: For scripts that perform modifications (add, update, delete, set), ALWAYS add `[CmdletBinding(SupportsShouldProcess)]` and wrap modification commands in `if ($PSCmdlet.ShouldProcess($target, $action))` blocks. This enables `-WhatIf` and `-Confirm` parameters for safe testing.
   - Use descriptive `$target` (e.g., field name, site URL) and `$action` (e.g., "Pin field to filter pane", "Delete file").
   - In `end` block summaries, ensure counters reflect intended operations (not just actual executions) when in WhatIf mode.
   - Example: `if ($PSCmdlet.ShouldProcess($fieldName, 'Pin field to filter pane')) { m365 spo field set ... }`
 - **begin/process/end blocks**:
   - `begin`: Login, validation, initialize script-level variables (e.g., `$script:ReportCollection`).
   - `process`: ALL modification commands (add, update, delete, publish) wrapped in `if ($PSCmdlet.ShouldProcess(...))`.
   - `end`: Summary, CSV export, transcript. NO modifications.
 - `m365 login --ensure` in begin block (NO `--output` flag). Check `$LASTEXITCODE`, throw on failure.
 - Long-form options (`--url` not `-u`), use `--output json` for parsing.
- Keep CLI invocations as single-line commands unless dynamic assembly needed.
- **NEVER use `Invoke-Expression`** to run CLI commands. Use direct command calls. Invoke-Expression is a security risk and makes code harder to read/debug.
 - **Direct command calls**: Use `m365 spo list get --url $SiteUrl --title "Documents" --output json` NOT `Invoke-Expression "m365 spo list get ..."`.
 - **Dynamic options (rare)**: If you must build commands dynamically, use arrays with splatting: `$args = @('spo','list','get','--url',$SiteUrl); if ($Condition) { $args += '--withPermissions' }; m365 @args`. Avoid string concatenation.
- Convert JSON: `@($json | ConvertFrom-Json)`.

## Performance Optimization
**Minimize API requests**:
 - **Filter by HasUniqueRoleAssignments**: Only items with unique permissions can have sharing links/permissions. Use `--fields "...,HasUniqueRoleAssignments"` + client-side filter to skip 90% of items.
 - **Prefer server-side filtering**: Use `--filter` (OData) or `--query` (JMESPath) over PowerShell `Where-Object` when possible.
   - `--filter`: OData queries like `"Hidden eq false"` or `"BaseTemplate eq 101"`
   - `--query`: JMESPath expressions like `"[?Hidden == \`false\` && BaseTemplate == \`101\`]"` for complex conditions. Common patterns:
     - Contains: `--query "[?contains(FileRef, 'folder')]"` (case-sensitive substring match)
     - Multiple conditions: `--query "[?contains(FileRef, 'folder') && FSObjType == \`0\`]"` (files only in folder)
     - **CRITICAL**: ALWAYS prefer `--query` over PowerShell `| Where-Object` for filtering CLI results. Use `Where-Object` ONLY when JMESPath cannot express the condition (e.g., `-like` wildcard patterns, regex).
   - Example: `m365 spo list list --query "[?!(contains(['Form Templates','Style Library','Site Pages'], Title))]"` filters out system libraries server-side
 - Use `--fields` to limit columns.

## Output Path Validation
 - Only validate `$OutputPath` in `begin` block if user explicitly specified it (not when using default `(Get-Location).Path`)
 - Pattern: `if ($PSBoundParameters.ContainsKey('OutputPath')) { Test-Path validation }`

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
  - **Date verification**: Run `date +%Y-%m-%d` BEFORE updating sample.json. Verify `updateDateTime` matches.
  - **CLI version verification**: Run `cat ../cli-microsoft365/package.json | grep '\"version\"'` to confirm exact version. Check sample.json uses this value.
  - Verify Adam's author entry: `grep -A3 'Adam-it' scripts/.../assets/sample.json` matches reference format above.
  - Check no Unicode escapes: `grep '\\u00' assets/sample.json` should return ZERO.
  - **JSON Formatting**: Verify consistent 2-space indentation. Run `python3 -c "import json; json.load(open('assets/sample.json'))"` to validate.

**Script**:
  - **NO comment blocks above script**: Check README.md does NOT have `.SYNOPSIS`, `.DESCRIPTION`, `.PARAMETER`, `.EXAMPLE` blocks above the PowerShell script. Examples should be at END of script, inside code block, commented with `#`.
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

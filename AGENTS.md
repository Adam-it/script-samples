# CLI for Microsoft 365 Script Implementation Guide

## Core Flow
1. **Research CLI commands**: Check `../cli-microsoft365/docs/docs/cmd/` to verify all commands exist with required options.
   - Avoid `m365 request` unless no specific command exists.
   - Use `--listId` or `--listUrl` over `--listTitle` (prevents confirmation prompts).
   - Filter hidden lists: `--filter "Hidden eq false"` excludes system lists.
   - Verify every command/option against docs during self-review.

## Metadata (sample.json)
   - Update `updateDateTime` (today) and CLI version from `../cli-microsoft365/package.json`.
   - Add `CLI-FOR-MICROSOFT365` metadata entry.
   - Add Adam Wójcik to `authors` (`gitHubAccount`: `Adam-it`).
   - Add CLI commands to `tags` (unique values only).
   - Add CLI reference; keep existing PnP references.

## README
   - Add CLI tab alongside PnP tab (preserve existing content).
   - Update summary to mention CLI.
   - Add Adam to Contributors table.
   - **Tab order**: CLI/PnP tabs BEFORE `## Contributors`. Verify: `grep -n '^# \[' README.md`

## Script Structure
 - `[CmdletBinding(SupportsShouldProcess)]` for destructive operations.
 - Typed parameters with `[Parameter(Mandatory/HelpMessage)]` attributes.
 - **begin/process/end blocks**:
   - `begin`: Login, validation, data loading.
   - `process`: ALL modification commands (add, update, delete, publish) wrapped in `if ($PSCmdlet.ShouldProcess(...))`.
   - `end`: Summary, export, transcript. NO modifications.
 - `m365 login --ensure` in begin block (NO `--output` flag). Check `$LASTEXITCODE`, throw on failure.
 - Long-form options (`--url` not `-u`), use `--output json` for parsing.
 - Use `--query` (JMESPath) for server-side filtering when possible.
 - Keep CLI invocations as single-line commands unless dynamic assembly needed.
 - Convert JSON: `@($json | ConvertFrom-Json)`.

## Performance Optimization
**Minimize API requests**:
 - **Filter by HasUniqueRoleAssignments**: Only items with unique permissions can have sharing links/permissions. Use `--fields "...,HasUniqueRoleAssignments"` + client-side filter to skip 90% of items.
 - Use `--filter` (OData) or `--query` (JMESPath) over `Where-Object`.
 - Use `--fields` to limit columns.
 - Use batch commands when available.

**Example**:
```powershell
# Good: Filter items with unique permissions first
$items = m365 spo listitem list --webUrl $url --listId $id --fields "FileRef,HasUniqueRoleAssignments" --output json | ConvertFrom-Json
$items = $items | Where-Object { $_.HasUniqueRoleAssignments -eq $true }
foreach ($item in $items) {
    $links = m365 spo file sharinglink list --webUrl $url --fileUrl $item.FileRef --output json
}
```

## Error Handling
 - `begin`: Use `throw` for critical errors (login, invalid paths).
 - `process`: Use `try/catch` with `Write-Warning` + `continue`. Never `return` or `throw` in loops.
 - Track failures: `$script:Summary.Failures++`.
 - Display failure count in `end` block with color-coding (red if > 0).

## Output & UX
 - **Progress**: Use `Write-Verbose` for progress; `Write-Host` with colors only in `end` block summaries.
 - **CSV Export**: Optional `[switch]$ExportToCsv` with `$OutputPath`. Export in `end` block. Use `($array | Where-Object { $_ }) -join '|'` for multi-value fields.
 - **Use WhatIf, not custom ReportOnly**: Rely on PowerShell's built-in `-WhatIf` support via `ShouldProcess`. Do NOT add custom `[switch]$ReportOnly` parameters.
 - **Transcript Logging**: Add `Start-Transcript` in `begin`, `Stop-Transcript` in `end`.
 - **Usage Examples**: Add 3-4 commented examples at END of script (not comment-based help at top). Always include WhatIf example, show basic usage, Verbose output.

## Self-Review
  - Verify all commands against docs.
  - Score honestly (start at 6-7, not 9-10).
  - Mark complete in plan.md with score, features, gaps.
  - **Validate README structure**: Run `grep -n '^# \[' README.md` to ensure no duplicate tab markers.

### README Structure Validation
After completing script implementation, verify structure with these commands:

```bash
# Check for duplicate tab markers - should show exactly 2 lines
grep -n '^# \[' scripts/<script-name>/README.md

# Verify tabs appear before Contributors
grep -n '^## Contributors' scripts/<script-name>/README.md
```

**Expected structure**:
```
## Summary
### Prerequisites
# [CLI for Microsoft 365](#tab/cli-m365-ps)
[CLI script]
[!INCLUDE [More about CLI...]]
# [PnP PowerShell](#tab/pnpps)
[PnP script]
[!INCLUDE [More about PnP...]]
***
## Source Credit
## Contributors
[!INCLUDE [DISCLAIMER]]
<img src="..." />
```

**Common mistakes**:
- ❌ Duplicate `# [CLI for Microsoft 365]` or `# [PnP PowerShell]` markers
- ❌ Script content outside code blocks (between tab markers)
- ❌ Missing `***` separator before Source Credit
- ❌ Tab markers after Contributors section

## Quick Checklist
- sample.json: date, CLI version 11.2.0, metadata, authors, tags (no duplicates).
- CLI tab follows structure; PnP tab untouched.
- Long-form CLI options; `--output json` for parsing.
- `ShouldProcess` wraps destructive operations.
- Check `$LASTEXITCODE` after CLI commands.
- Usage examples at end (not top), include WhatIf example.
- Mark complete in plan.md.

## Guardrails
- **NO Python scripts** for simple file edits. Use `apply_patch` directly.
- Never remove PnP content.
- No credentials/tenant info in samples.
- No backslash escaping: `$variable` not `\\$variable`.

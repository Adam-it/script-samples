# CLI for Microsoft 365 Script Implementation Guide

## Core Flow
1. **Verify scenario**: Confirm if refactoring existing CLI tab or adding new one alongside PnP PowerShell.
2. **Research CLI commands**: Check `../cli-microsoft365/docs/docs/cmd/` and `../cli-microsoft365/allCommands.json` to find appropriate commands.
   - **Avoid `m365 request`**: Only use as last resort when no specific CLI command exists.
   - **Use unique identifiers**: When working with SharePoint lists/libraries, always prefer `--listId` or `--listUrl` over `--listTitle`. Multiple lists can have the same title, causing CLI commands to fail or prompt for confirmation (breaking automation). Use `Id` or `Url` properties which are guaranteed unique.
   - During self-review, confirm every CLI command and option against docs.

## Metadata (sample.json)
   - Update `updateDateTime` (today) and CLI version from `../cli-microsoft365/package.json`.
   - Add or update `CLI-FOR-MICROSOFT365` entry in `metadata`.
   - Keep PnP references; add the CLI reference when introducing a CLI tab.
   - Add Adam Wójcik to `authors` (`gitHubAccount`: `Adam-it`).
   - Add CLI commands to `tags` (unique values only, no duplicates).
   - **Validate JSON**: `python3 -m json.tool <file>` to catch syntax errors (extra commas, missing commas).

## README Updates
   - Preserve PnP tab; add CLI tab.
   - Update summary to mention both PnP and CLI.
   - Add Adam to Contributors table.

## Script Structure
  - Advanced function: `[CmdletBinding(SupportsShouldProcess)]` for destructive operations.
  - Typed parameters with `[Parameter(Mandatory/HelpMessage)]` attributes.
  - `begin/process/end` blocks.
  - `m365 login --ensure` in the begin block (no `--output` flag), verify login by checking `$LASTEXITCODE` immediately after the command and `throw` on failure.
  - Long-form CLI options (`--url` not `-u`), use `--output json` for parsing.
   - **JMESPath Filtering**: Use `--query` to filter results server-side instead of PowerShell `Where-Object` when possible:
     - Reduces memory usage and improves performance (filtering happens before JSON parsing)
     - Syntax: `--query "[?property == 'value']"` or `--query "[?property == \`$true\`]"` for booleans
     - Escape backticks in PowerShell: `` --query "[?Active == \`$true\`]" ``
     - Common patterns:
       - Boolean filter: `--query "[?HasUniqueRoleAssignments == \`$true\`]"`
       - String filter: `--query "[?Title == 'Documents']"`
       - Nested property: `--query "[?link.scope == 'anonymous']"`
       - Multiple conditions: `--query "[?Active == \`$true\` && Status == 'Approved']"`
     - **When NOT to use**: Complex PowerShell logic (e.g., `-notin`, regex, custom comparisons) — filter in PowerShell instead.
     - See [JMESPath Tutorial](http://jmespath.org/tutorial.html) for advanced syntax.
  - Keep CLI invocations as readable single-line commands unless dynamic option assembly is unavoidable.
  - Convert JSON with `@($json | ConvertFrom-Json)`.

## Multi-tenant Support
   - CLI for Microsoft 365 supports multiple simultaneous connections:
     - Use `m365 login --connectionName <name>` to create a named connection when logging in to different tenants
     - Use `m365 connection use --name <name>` to switch the active connection
     - Each connection maintains its own auth state (no re-authentication when switching).

## Error Handling
     - **`begin` block**: Use `throw` for critical errors (login, invalid paths).
     - **`process` block**: Use `try/catch` with `Write-Warning` and `continue`. Never use `return` or `throw` inside loops.
     - Track failures in `$script:Summary.Failures++`.
     - Always display failure count in the `end` block summary with color-coding (red if > 0).

## Output & UX
   - **Progress messages**: Use `Write-Verbose` for progress; `Write-Host` with colors only in `end` block for summaries.
   - **CSV Export**: Optional `[switch]$ExportToCsv` with `$OutputPath` parameter. Export in `end` block; otherwise display with `Format-Table`.
   - **Report-Only Mode**: For destructive operations, add `[switch]$ReportOnly` that shows what would be affected (titles, URLs, counts) without performing actions. Complements `-WhatIf` with richer preview.
   - **Usage Examples**: Add 3-4 commented usage examples at the end of the script (not comment-based help at the top, per user preference). Examples should demonstrate: basic usage, report-only mode (if applicable), WhatIf mode, and verbose output. Keep examples concise and practical.

## Self-Review
  - Score CLI + PowerShell practices (0–10) with strengths and improvements.
  - Suggest future enhancements: performance optimizations, additional parameters, edge cases.
  - **BE HONEST AND CRITICAL**: Start with lower scores (5-6) if uncertain. Do not inflate scores.
  - **ALWAYS VERIFY**: Check every CLI command and option against documentation in `cli-microsoft365/docs/` folder.
  - **Common pitfalls**:
    - Assuming CLI options exist without verification (e.g., `--url` to rename lists)
    - Over-engineering with unnecessary command building patterns
    - Adding options at the end when they could be inline (e.g., `--output json` position)
    - Not testing command assumptions against actual CLI behavior

### README Tab Order Self-Check (CRITICAL)

Before completing any script implementation, **ALWAYS verify README.md tab structure**:

1. **Verify tab order** matches this pattern:
   ```
   ## Summary
   ### Prerequisites
   # [CLI for Microsoft 365](#tab/cli-m365-ps)
   [CLI script content]
   [!INCLUDE [More about CLI for Microsoft 365](../../docfx/includes/MORE-CLIM365.md)]
   # [PnP PowerShell](#tab/pnpps)
   [PnP script content]
   [!INCLUDE [More about PnP PowerShell](../../docfx/includes/MORE-PNPPS.md)]
   ***
   ## Source Credit
   ## Contributors
   [!INCLUDE [DISCLAIMER](../../docfx/includes/DISCLAIMER.md)]
   <img src="..." />
   ```

2. **Common mistakes to check**:
   - ❌ CLI tab placed AFTER Contributors section
   - ❌ CLI tab placed AFTER disclaimer or telemetry img tag
   - ❌ Missing `***` separator before Source Credit
   - ❌ Duplicate tab markers (e.g., two `# [PnP PowerShell]` lines)
   - ❌ Script content appearing outside tab sections

3. **Self-review checklist**:
   - [ ] CLI/PnP tabs appear BEFORE `## Source Credit`
   - [ ] Contributors section is AFTER all script tabs
   - [ ] No content between disclaimer and telemetry img
   - [ ] Tab order is consistent (CLI → PnP or PnP → CLI)
   - [ ] `***` separator closes all tabs before Source Credit

4. **Quick verification command**:
   ```bash
   grep -n "^# \[" scripts/<script-name>/README.md
   grep -n "^## Contributors" scripts/<script-name>/README.md
   ```
   Tab markers (`# [CLI` or `# [PnP`) should have **lower line numbers** than `## Contributors`.

## Quick Checklist
- ✅ sample.json: date, version, metadata, authors, references updated. Validate with `python3 -m json.tool`.
- ✅ Tags: no duplicates, match actual commands.
- ✅ CLI README tab follows best practices; PnP tab untouched.
- ✅ Long-form CLI options; use `--output json` and `--query` for filtering.
- ✅ `ShouldProcess` protects destructive operations; no unexpected prompts.
- ✅ Check `$LASTEXITCODE` after CLI commands.
- ✅ Usage examples at end of script (3-4 examples).
- ✅ Mark complete in plan.md.

## Guardrails
- No throwaway helper scripts for tiny edits.
- Never remove PnP content; mimic its behaviour.
- Keep credentials/tenant info out of samples.
- No backslash escaping: use `$variable` not `\$variable` in README scripts.

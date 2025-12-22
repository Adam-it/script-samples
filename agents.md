# CLI Script Agent Cheatsheet

## Core Flow
1. **Recap scenario**: confirm if you are refactoring an existing CLI tab or adding a new one alongside PnP PowerShell.
2. **Plan & research**: outline inputs/outputs, check CLI docs in `../cli-microsoft365/docs/docs/cmd/`, and map auth → actions → reporting.
   - Review `../cli-microsoft365/allCommands.json` before defaulting to `m365 request` so you pick the most appropriate built-in command.
   - During self-review, confirm every CLI command and option in your script against both `allCommands.json` and the relevant docs page to ensure arguments are valid.
   - **Use unique identifiers**: When working with SharePoint lists/libraries, always prefer `--listId` or `--listUrl` over `--listTitle`. Multiple lists can have the same title, causing CLI commands to fail or prompt for confirmation (breaking automation). Use `Id` or `Url` properties which are guaranteed unique.
3. **Metadata touch-up (sample.json)**:
   - Update `updateDateTime` and CLI version (from `../cli-microsoft365/package.json`).
   - Append or update the `CLI-FOR-MICROSOFT365` entry in `metadata`; never invent new keys.
   - Keep PnP references; add the CLI reference when introducing a CLI tab.
   - Extend `authors` with Adam Wójcik (`gitHubAccount` `Adam-it`, picture URL) when you touch the CLI sample.
   - Adjust `tags` only after the script is final so they match the actual commands. The `tags` array must contain unique values only - no duplicates. Each command from both PnP PowerShell and CLI for Microsoft 365 should appear exactly once.
   - **Validate JSON syntax**: After making changes, ensure the JSON is valid. Common mistakes:
     - Extra commas after closing braces (e.g., `}},` should be `},`)
     - Missing commas between array items (e.g., in `tags` array)
     - Verify with `python3 -m json.tool <file>` to catch syntax errors
4. **README updates**:
   - Preserve the PnP tab.
   - New CLI tabs must mirror PnP inputs/outputs, update the summary, and include Adam in the contributors list.
5. **Author the CLI script**:
   - Advanced function with `CmdletBinding`, typed params with `HelpMessage` for each parameter (provides user guidance), `begin/process/end` blocks.
   - `m365 login --ensure` in the begin block (no `--output` flag), verify login by checking `$LASTEXITCODE` immediately after the command and `throw` on failure.
   - Long-form options, handle output with `--output json` and `--query` for filtering.
   - Keep CLI invocations as readable single-line commands unless dynamic option assembly is unavoidable.
   - Convert CLI JSON results with native `@($json | ConvertFrom-Json)` instead of custom helpers; stick to arrays so summaries can use `+=`.
   - **Avoid `m365 request` command**: Only use `m365 request` as a last resort when absolutely no specific CLI for Microsoft 365 command exists for a scenario. Using `m365 request` should be considered a temporary workaround until a dedicated CLI command is implemented. If a script heavily relies on REST API calls via `m365 request`, consider selecting a different script to implement that uses specific CLI commands instead.
   - **Multi-tenant/Multi-connection Support**: CLI for Microsoft 365 supports managing multiple connections simultaneously, which is essential for cross-tenant operations:
     - Use `m365 login --connectionName <name>` to create a named connection when logging in to different tenants
     - Use `m365 connection list --output json` to list all available connections
     - Use `m365 connection use --name <name>` to switch the active connection
     - Example workflow for comparing files across tenants:
       1. `m365 login --connectionName "tenant1"` (authenticate to first tenant)
       2. `m365 login --connectionName "tenant2"` (authenticate to second tenant)
       3. `m365 connection use --name "tenant1"` (switch to first tenant)
       4. Execute commands against first tenant
       5. `m365 connection use --name "tenant2"` (switch to second tenant)
       6. Execute commands against second tenant
     - Each connection maintains its own authentication state, so you can switch between tenants without re-authenticating
     - Connection names can be custom (e.g., "myworkaccount", "tenant1") or auto-generated GUIDs
   - **Error Handling Best Practices**:
     - Use `throw` for critical errors in `begin` block (login failures, invalid paths, missing prerequisites). Simpler than `Write-Error` + `exit 1` and provides better stack traces.
     - Per-item errors in `process` block: use `try/catch` with `Write-Warning` and `continue` to process remaining items. Never use `return` or `throw` inside loops as it exits the entire script.
     - Track all failures in the summary counter (`$script:Summary.Failures++`) and optionally add failed items to the report with error details so users can see what failed in the CSV export.
     - Always display failure count in the `end` block summary with color-coding (red if > 0).
   - **Use Write-Verbose for progress messages**: Informational messages about script progress (e.g., "Retrieving groups...", "Found X items...") should use `Write-Verbose` so users can control verbosity with `-Verbose` flag. Reserve `Write-Host` with colors only for final status messages in the `end` block (e.g., success/warning summaries).
   - **CSV Export Best Practice**: Use an optional `[switch]$ExportToCsv` parameter with an optional `$OutputPath` parameter (default to current directory). Perform CSV export in the `end` block as a summary action. If the switch is not specified, display results in the terminal using `Format-Table`. This keeps scripts flexible for both automation (CSV) and interactive use (terminal output).
   - **Comment-based help with examples**: Always include comprehensive comment-based help (`.SYNOPSIS`, `.DESCRIPTION`, `.PARAMETER`, `.EXAMPLE`) at the start of the script. Include at least 3-4 examples showing different usage patterns, with the most complex example including the `-Verbose` flag to demonstrate comprehensive feedback.
6. **Self-review**: run the checklist, note CLI + PowerShell scores (0–10) with strengths and improvement ideas, then pause.
   - **Present improvement suggestions**: After completing the script, include a section in your summary that suggests potential future enhancements or alternative approaches. Examples:
     - Performance optimizations for large tenants (e.g., parallel processing, batching)
     - Additional parameters that could make the script more flexible
     - Alternative CLI commands or approaches that could achieve the same goal
     - Edge cases that might need special handling
     - Integration possibilities with other scripts or workflows
   - These suggestions help demonstrate forward thinking and provide the user with a roadmap for future enhancements.

## Quick Checklist
- ✅ sample.json valid, date/version/metadata/authors/references updated (no extra keys).
  - Verify with `python3 -m json.tool scripts/<folder>/assets/sample.json` before marking complete.
  - Ensure `tags` array has no duplicate entries.
  - Check for syntax errors: extra commas (`}},`), missing commas between items.
- ✅ CLI README tab follows best practices; PnP tab untouched.
- ✅ Tags align with the final CLI commands.
- ✅ Long-form CLI options and server-side filtering (`--query`) wherever supported.
- ✅ `ShouldProcess` protects destructive operations; no unexpected prompts.
- ✅ Only intentional files changed; formatting clean.

## Guardrails
- No throwaway helper scripts for tiny edits.
- Never remove PnP content; mimic its behaviour.
- Keep credentials/tenant info out of samples.
- Prefer `Write-Verbose`/`Write-Warning`; summaries belong in the `end` block.

## PowerShell Syntax Validation

When adding PowerShell scripts to README files:
- **NEVER** escape `$` with backslashes (`\$`) - this breaks PowerShell syntax and makes variables invalid.
- Verify variables use correct syntax: `$variable`, not `\$variable`.
- String interpolation: `"text $($var.property)"`, not `"text \$(\$var.property)"`.
- After adding a script to README, verify PowerShell syntax visually or extract and validate with PowerShell parser if possible.
- Test that comment-based help renders correctly (`. .\script.ps1; Get-Help .\script.ps1`).
- Common issues to watch for:
  - Extra backslashes before `$` symbols
  - Incorrect array syntax
  - Missing or extra braces/brackets
  - Invalid parameter attributes

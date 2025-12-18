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
   - Adjust `tags` only after the script is final so they match the actual commands.
4. **README updates**:
   - Preserve the PnP tab.
   - New CLI tabs must mirror PnP inputs/outputs, update the summary, and include Adam in the contributors list.
5. **Author the CLI script**:
   - Advanced function with `CmdletBinding`, typed params with `HelpMessage` for each parameter (provides user guidance), `begin/process/end` blocks.
   - `m365 login --ensure` in the begin block (no `--output` flag), verify login by checking `$LASTEXITCODE` immediately after the command and throw error if it fails.
   - Long-form options, handle output with `--output json` and `--query` for filtering.
   - Keep CLI invocations as readable single-line commands unless dynamic option assembly is unavoidable.
   - Convert CLI JSON results with native `@($json | ConvertFrom-Json)` instead of custom helpers; stick to arrays so summaries can use `+=`.
   - Wrap CLI calls, check `$LASTEXITCODE`, record successes/failures, add end-of-run summary, support `ShouldProcess`/`WhatIf`.
   - **Export to CSV when appropriate**: Consider adding an optional `$ExportCsvPath` parameter to export summary or detailed results to CSV in the `end` block. This helps with reporting and auditing.
   - Example usage comment at the end should include `-Verbose` flag to demonstrate comprehensive feedback.
6. **Self-review**: run the checklist, note CLI + PowerShell scores (0–10) with strengths and improvement ideas, then pause.

## Quick Checklist
- ✅ sample.json valid, date/version/metadata/authors/references updated (no extra keys).
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

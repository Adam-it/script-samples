# CLI Script Agent Cheatsheet

## Core Flow
1. **Recap scenario**: confirm if you are refactoring an existing CLI tab or adding a new one alongside PnP PowerShell.
2. **Plan & research**: outline inputs/outputs, check CLI docs in `../cli-microsoft365/docs/docs/cmd/`, and map auth → actions → reporting.
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
   - Advanced function with `CmdletBinding`, typed params + `HelpMessage`, `begin/process/end` blocks.
   - `m365 login --ensure`, long-form options, handle output with `--output json` and `--query` for filtering.
   - Keep CLI invocations as readable single-line commands unless dynamic option assembly is unavoidable.
   - Convert CLI JSON results with native `@($json | ConvertFrom-Json)` instead of custom helpers; stick to arrays so summaries can use `+=`.
   - Wrap CLI calls, check `$LASTEXITCODE`, record successes/failures, add end-of-run summary, support `ShouldProcess`/`WhatIf`.
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

# CLI Script Agent Cheatsheet

## Core Flow
1. **Recap scenario**: read the README to understand what the sample does and whether you are refactoring an existing CLI tab or creating a brand-new one alongside the PnP PowerShell version.
2. **Plan & research**: review required inputs/outputs, skim the CLI docs in `../cli-microsoft365/docs/docs/cmd/` for the exact commands and options, and map authentication → data gathering → actions → reporting before touching files.
3. **Metadata touch-up (sample.json)**:
   - Update `updateDateTime` to today and set the CLI version from `../cli-microsoft365/package.json`.
   - Keep existing references and append the CLI for Microsoft 365 reference when adding a new tab; never remove the PnP reference.
   - Add Adam Wójcik ("Adam-it") to `authors` whenever you touch the CLI sample.
   - Leave `tags` until the script is final, then list only the CLI commands actually used.
4. **README updates**:
   - Preserve the PnP PowerShell tab.
   - For refactors: adjust the existing CLI tab in place.
   - For new CLI tabs: add a section that mirrors the PnP version’s inputs, outputs, and behaviour (e.g., report files, parameter purpose). Update the sample description/introduction and author list to mention the CLI support.
5. **Author the CLI script**:
   - Use an advanced function (`function Name { [CmdletBinding(SupportsShouldProcess = ...)] param(...) ... }`).
   - Provide a meaningful `param()` block with explicit types and concise `HelpMessage` text for each parameter, aligning with the PnP script’s inputs when possible.
   - Structure logic with `begin`, `process`, and `end` blocks: handle login and setup in `begin`, main work in `process`, and summarise/report in `end`.
   - Call `m365 login --ensure`, prefer long-form option names, capture output with `2>&1`, check `$LASTEXITCODE`, and use `--output json`.
   - Push filtering into the CLI with `--query` (JMESPath) instead of `Where-Object` whenever the command supports it; only fall back to PowerShell filtering if the CLI response cannot be narrowed server-side.
   - Wrap CLI calls in try/catch, surface errors with helpful messages, and continue processing when possible.
   - Avoid interactive prompts; rely on `ShouldProcess`/`WhatIf` and optional `-Force` switches for destructive actions.
6. **Self-review**: walk this checklist slowly, ensure formatting is clean, record 0–10 scores for CLI usage and PowerShell practices (with strengths and improvement ideas), then stop unless more work is requested.

## Quick Checklist
- ✅ sample.json updated (date, version, references, authors) and valid JSON.
- ✅ README CLI section follows best practices; PnP tab untouched.
- ✅ Tags reflect final CLI commands (done last).
- ✅ Long-form CLI options and server-side filtering (`--query`) where possible.
- ✅ `ShouldProcess` guards changes; no unexpected prompts.
- ✅ Only intentional files modified.
- ✅ Example usage provided when helpful.

## Guardrails
- No ad-hoc helper scripts for tiny edits—modify files directly.
- Never remove PnP content; mirror its behaviour with CLI where practical.
- Keep tenant-specific data, credentials, and secrets out of samples.
- Prefer `Write-Verbose`/`Write-Warning` over `Write-Host`; reserve `Write-Host` for final summaries if needed.
- Summaries live in the `end` block.

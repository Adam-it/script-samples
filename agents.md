# CLI Script Agent Cheatsheet

## Core Flow
1. **Recap scenario**: read README, note CLI commands needed, decide if adding or refactoring.
2. **Plan & research**: skim docs under `../cli-microsoft365/docs/docs/cmd/`, sketch auth → inputs → actions → outputs.
3. **Metadata touch-up**: update `updateDateTime`, set CLI version from `../cli-microsoft365/package.json`, add Adam Wójcik if the CLI tab changed, keep PnP reference, and fix tags *after* the script is final.
4. **Write the CLI tab**: advanced function with `CmdletBinding`, meaningful `param()` block (each parameter has a concise `HelpMessage`), `begin/process/end`, `m365 login --ensure`, prefer long-form option names for readability, JSON output, `--query` where possible, `ShouldProcess` for changes, end-of-run summary.
5. **Self-review**: walk the checklist below slowly, then record CLI and PowerShell scores (0–10) with strengths + improvement ideas in your hand-off.

## Checklist (tick mentally)
- Metadata updated & valid JSON.
- README CLI tab matches best practices (params, login, error handling, summary).
- Tags reflect actual commands used (done last) and Adam Wójcik is added if you touched the CLI sample.
- Diff only shows intentional changes.
- Avoid prompts in default execution paths—use `ShouldProcess` for visibility and `Force` only when bypassing confirmations is acceptable.
- Honour the current task scope: when the user limits the work to a specific step, complete that step and pause for confirmation before moving on.

## Quick Rules
- No temporary helper scripts for tiny edits.
- Keep original PnP tab; add CLI alongside it.
- Avoid hard-coded tenant details or credentials.
- Use `ConvertFrom-Json` inside try/catch and check `$LASTEXITCODE` after each CLI call.
- Prefer server-side filtering (`--query`) before PowerShell filtering.
- Always respect `ShouldProcess` for create/update/delete actions.

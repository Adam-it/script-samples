# Agent Playbook: CLI for Microsoft 365 Script Samples

## Mission
Create and maintain script samples in this repository that showcase best-practice usage of CLI for Microsoft 365 within PowerShell. Deliver scripts that are tenant-ready, well-documented, and aligned with the latest CLI release.

## Recommended Workflow
1. **Initial Recon**
   - Read the existing README and script (if any) to understand the scenario.
   - Identify SharePoint/Entra/Teams touchpoints and required CLI commands.
   - Determine whether you are adding a brand-new CLI sample or refactoring an existing one.

2. **Research & Planning**
   - Consult CLI docs in `../cli-microsoft365/docs/docs/cmd/<product>/` for syntax, options, and `--query` usage.
   - Prefer commands that support JSON output and JMESPath filtering to minimise local parsing.
   - Draft the sequence of CLI calls and PowerShell logic (auth → inputs → retrieval → processing → reporting).

3. **Update Metadata (`assets/sample.json`)**
   - Bump `updateDateTime` to today.
   - Ensure `CLI-FOR-MICROSOFT365` metadata exists with the latest version (check `../cli-microsoft365/package.json`).
   - Add yourself (Adam Wójcik / Adam-it) to authors if you made significant CLI changes.
   - Extend `tags` to include each CLI command used (e.g., `m365 spo list get`).
   - Keep existing PnP PowerShell references and add a **single** CLI reference entry (typically `https://aka.ms/cli-m365`) rather than one per command.
   - Skip `m365 spo set --url` for SPO workflows; `m365 login --ensure` followed by any SPO command automatically targets the tenant admin center.

4. **Author the CLI Script (README tab)**
   - Use an advanced function with `CmdletBinding`, a `param()` block, and `begin/process/end` sections.
   - Define parameters with validation, types, mandatory flags, and helpful `HelpMessage` text.
   - Authenticate via `m365 login --ensure` (no forced authtype or redundant output).
   - Request data with `--output json` and leverage `--query` to filter on the server side.
   - Handle errors: check `$LASTEXITCODE`, wrap `ConvertFrom-Json` in try/catch, and emit actionable messages.
   - Support `ShouldProcess`/`-WhatIf` for destructive operations; keep scripts idempotent when possible.
   - Provide user feedback (`Write-Verbose`, `Write-Host`) and summary statistics in the `end {}` block.
   - Keep CLI invocations readable (single-line commands when feasible).

5. **Quality Review**
   - Validate Markdown: blank line after headings, fenced code blocks with language hint (` ```powershell`).
   - Ensure parameters have meaningful defaults or are mandatory; avoid hard-coded tenant data.
   - Double-check CLI call count: combine steps using `--query` instead of separate calls when possible.
   - Verify summary counters align with collected data (processed/succeeded/failed).
   - Run `git diff` to confirm only intentional changes remain.
   - Score the update (0-10) for CLI usage practices and PowerShell quality; call out strengths and improvement ideas in the handoff.

6. **Documentation & Handoff**
   - Ensure both PnP and CLI tabs remain in the README (unless the scenario is CLI-only).
   - Reference relevant CLI docs in `sample.json` if new commands are introduced.
   - Mention test guidance (run with `-WhatIf`, validate in non-production tenant) when appropriate.

## Helpful References
- CLI for Microsoft 365 docs: `../cli-microsoft365/docs/docs/cmd/` (search for product-specific command pages).
- Prior refactors: see `create-dummy-docs-in-library`, `spo-sharepoint-alerts-audit`, `spo-update-document-library-templates` for pattern examples.
- Root repo README for formatting conventions.

## Do / Don’t Checklist
- ✅ Use `--output json` + `ConvertFrom-Json` with error handling.
- ✅ Apply JMESPath `--query` filters to limit response payloads.
- ✅ Track outcomes (processed/succeeded/failed) and summarise at the end.
- ✅ Keep scripts parameter-driven and reusable.
- ❌ Don’t rely on temporary Python helpers for simple edits—modify files directly.
- ❌ Don’t drop existing PnP content; add CLI content alongside it.
- ❌ Don’t leave hard-coded tenant values or credentials in samples.

## Pre-Commit Sanity Check
- [ ] `assets/sample.json` updated, valid JSON, correct CLI version.
- [ ] README contains updated CLI tab with advanced function.
- [ ] Script honours PowerShell best practices (`CmdletBinding`, `ShouldProcess`, descriptive params).
- [ ] CLI commands tested or at least documented with dry-run guidance.
- [ ] `git diff` reflects clean, intentional changes only.

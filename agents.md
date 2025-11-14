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
   - Default to server-side JSON + JMESPath filtering (`--query`) whenever the command supports it; only fall back to PowerShell filtering when you document why `--query` is unsuitable.
   - Draft the sequence of CLI calls and PowerShell logic (auth → inputs → retrieval → processing → reporting).

3. **Update Metadata (`assets/sample.json`)**
   - Bump `updateDateTime` to today.
   - Ensure `CLI-FOR-MICROSOFT365` metadata exists with the latest version (check `../cli-microsoft365/package.json`).
   - Add yourself (Adam Wójcik / Adam-it) to authors if you made significant CLI changes.
   - Extend `tags` to include each CLI command used (e.g., `m365 spo list get`).
   - Keep existing PnP PowerShell references and add a **single** CLI reference entry (typically `https://aka.ms/cli-m365`).
   - Skip `m365 spo set --url`; `m365 login --ensure` + any SPO command targets the admin center automatically.

4. **Author the CLI Script (README tab)**
   - Use an advanced function with `CmdletBinding`, a `param()` block, and `begin/process/end` sections.
   - Keep authentication and setup in `begin`; run data retrieval and core processing in `process`; reserve `end` for reporting/cleanup.
   - Define parameters with validation, types, mandatory flags, and helpful `HelpMessage` text.
   - Authenticate via `m365 login --ensure` (no forced authtype or redundant output).
   - Follow the Quick Rules section for authentication patterns, output handling, summaries, etc.

5. **Quality Review**
   - Validate Markdown, ensure parameters are meaningful/mandatory, prefer `--query`, align outputs with existing tabs, confirm `git diff`, and record 0‑10 self-scores.

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

## Quick Rules
- Authenticate with `m365 login --ensure`; skip `m365 status`/`m365 logout`.
- Prefer single CLI calls with `--query`; only fall back to `Where-Object` when documented.
- Guard `ConvertFrom-Json` with try/catch and check `$LASTEXITCODE` each time.
- Use `ShouldProcess` for destructive actions and report summaries in `end {}`.
- Reuse naming/outputs from the original tab; only create directories when the scenario already did.

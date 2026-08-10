---
name: quality-team
description: >
  Runs a chain of general-purpose sub-agents to audit, plan, carefully refactor
  and document any codebase. The skill detects the stack at runtime, loads only
  the relevant playbooks, then spawns:
  scout → principles-auditor → refactor-executor → doc-updater.
  Use when: "audit", "code quality", "refactor", "dead code", "clean up the
  code", "too many structural problems" — and the French equivalents:
  "audit qualité", "nettoie le code", "code mort", "trop de problèmes
  structurels". Do not use: adding a feature, fixing one specific bug,
  generating tests from scratch.
when_to_use: >
  Trigger on: "quality-team", "audit", "refactor", "dead code", "code quality",
  "clean up the code", "structural problems" — and the French equivalents:
  "audit qualité", "nettoie le code", "code mort", "mauvaise qualité",
  "problèmes structurels".
  Do not trigger on: adding a feature, fixing a specific bug, writing tests.
argument-hint: "[scope] [mode]"
arguments:
  - scope     # relative path to analyse (default: .)
  - mode      # "audit-only" | "refactor" | "all" (default: all)
allowed-tools:
  - Read
  - Bash
user-invocable: true
---

## Role of the skill

You are the orchestrator. Stay thin: detect the context, load the references,
spawn the agents, check the output files and present the decisions to the user.
Do not carry the detailed audit logic in this file.

Read the scope and the mode from `$ARGUMENTS`.

- If `$ARGUMENTS` is empty: `scope=.`, `mode=all`
- If there is a single argument: it is the scope, `mode=all`
- If there are two arguments: first=scope, second=mode
- Valid modes: `audit-only`, `refactor`, `all`

Always replace `<scope>` and `<mode>` with their literal values in the prompts
sent to the sub-agents.

## Phase 0 — Preparation and detection

Create `.claude/quality-team` if it does not exist.

Detect the project without assuming a stack:
- manifests: `package.json`, `Cargo.toml`, `pyproject.toml`, `setup.py`,
  `requirements.txt`, `go.mod`, `pom.xml`, `build.gradle`, `Makefile`,
  `composer.json`, `Gemfile`, `.sln` / `.csproj` files
- dominant languages, from the extensions present in `<scope>`
- available validation commands: `test`, `lint`, `typecheck`, `check`, `build`
  scripts, or native commands whose manifest/config exists

Record:
- `.claude/quality-team/project_profile.json`
- `.claude/quality-team/validation_commands.json`
- `.claude/quality-team/baseline_validation.json`

If no validation is detected, record the validation as `skipped` and continue.

## Phase 0b — References

Read and keep available for the prompts:
- `references/principles.md`
- `references/safe-refactor.md`
- `references/ai-smells.md`
- `templates/refactor-plan.md`
- `references/clean-code-rules.md` if the context allows it
- `references/refactoring-rules.md` if the context allows it

Load the playbooks only if the project profile justifies them:
- `playbooks/react-ts.md` only when React / TypeScript / Tauri is detected
- `playbooks/rust.md` only when Rust is detected

An optional playbook must never produce a violation on a project that does not
match its stack.

## Phase 1 — Scout

Spawn `scout`:

```text
Analyse the scope: <scope>.
quality-team mode: <mode>.
Read .claude/quality-team/project_profile.json and validation_commands.json.
Produce: .claude/quality-team/findings.json
Constraint: read-only, no file modification.
```

Wait until it finishes. If `findings.json` is missing, stop with a clear message.

## Phase 2 — Principles auditor

Spawn `principles-auditor` with `findings.json`, the universal references and
the optional playbooks loaded in Phase 0b:

```text
Analyse the scope: <scope>.
quality-team mode: <mode>.
Read .claude/quality-team/findings.json.
Produce: .claude/quality-team/violations.json
Constraint: read-only, no file modification.
Apply the universal principles first, then only the applicable playbooks.
```

Wait until it finishes. If `violations.json` is missing, stop with a clear
message.

## Phase 2b — Plan and user approval

Build `.claude/quality-team/refactor_plan.md` from
`templates/refactor-plan.md`, `findings.json`, `violations.json` and
`validation_commands.json`.

Present the plan before any modification. In `refactor` and `all` modes, never
launch `refactor-executor` without an explicit approval from the user (`yes`,
`approved`, `go ahead`, `run it`, or their French equivalents `oui`, `valide`,
`continue`, `lance`). Any missing, ambiguous or negative answer switches the run
to report-only.

In `audit-only` mode, present the plan as recommendations and skip Phase 3.

## Phase 3 — Refactor executor

Only if `mode != audit-only` and the plan was approved, spawn
`refactor-executor`:

```text
Analyse the scope: <scope>.
quality-team mode: <mode>.
Plan approved by the user: yes.
Read .claude/quality-team/refactor_plan.md.
Read .claude/quality-team/violations.json.
Read .claude/quality-team/findings.json.
Read .claude/quality-team/validation_commands.json.
Produce: .claude/quality-team/changes.json
Apply only the changes planned in refactor_plan.md and allowed by safe-refactor.md.
Validate with the detected commands, if any exist.
If validation fails: revert the modified file and log it in changes.skipped.
```

## Phase 4 — Doc updater

Spawn `doc-updater`:

```text
Analyse the scope: <scope>.
quality-team mode: <mode>.
Read findings.json, violations.json, refactor_plan.md, validation_commands.json.
Read changes.json only if it exists.
Generate REFACTOR_REPORT.md at the root of the analysed project.
Never modify source code; only documentation may change.
```

## Phase 5 — Final summary

Re-run the detected validation commands if a refactoring was applied.
Compare `baseline_validation.json` with the post-refactoring results.

Display:
- scope and mode
- validation before → after for each detected command
- path to the report
- changes applied / skipped
- files left for manual verification
- playbooks loaded

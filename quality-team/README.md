# quality-team — Audit and careful refactoring for any codebase

`/quality-team` runs a chain of 4 general-purpose sub-agents. The skill detects
the project at runtime, loads the universal rules, then adds only the playbooks
matching the detected stack.

## Agent chain

| Agent | Role | Output |
|-------|------|--------|
| scout | Detects manifests, languages, tools, hotspots, complexity, dead code, clones and best-effort lint. | `findings.json` |
| principles-auditor | Applies the universal principles, the AI smells and the applicable playbooks. | `violations.json` |
| refactor-executor | Applies only the changes approved by the user and allowed by the safety rules. | `changes.json` |
| doc-updater | Produces the final report and updates project documentation only. | `REFACTOR_REPORT.md` |

## Files produced

```text
.claude/quality-team/
  project_profile.json       # manifests, languages, applicable playbooks
  validation_commands.json   # commands detected for this project
  baseline_validation.json   # state before refactoring
  findings.json              # consolidated static analysis
  violations.json            # classified violations
  refactor_plan.md           # plan presented before any modification
  changes.json               # changes applied/skipped (absent in audit-only)

REFACTOR_REPORT.md           # readable report, at the root of the analysed project
```

## Usage

```text
/quality-team .
/quality-team src audit-only
/quality-team packages/api refactor
```

In `refactor` and `all` modes, the skill always presents a plan before any
modification. Refactoring starts only after explicit approval.

## General-purpose by default

The core depends on no stack. It can analyse a JavaScript, TypeScript, Rust,
Python, Go, Java, PHP, Ruby, .NET or mixed project as long as source files
exist. If no validation command is detected, the pipeline carries on and marks
the validation as `skipped`.

## Optional playbooks

Playbooks add specialised rules only when the project justifies them:

- `playbooks/react-ts.md`: React / TypeScript / Tauri detected
- `playbooks/rust.md`: Rust detected

A playbook that does not apply must never produce a violation.

## Limitations

- Files without enough context, or too risky to touch, are placed in `manual_verify`.
- Automatic fixes are deliberately kept small.
- Validations depend on the scripts and tools present in the target project.
- Qartez improves precision considerably, but the skill works without any MCP.

## Architecture

```text
quality-team/
  SKILL.md
  references/
    principles.md
    safe-refactor.md
    ai-smells.md
    clean-code-rules.md
    refactoring-rules.md
  playbooks/
    react-ts.md
    rust.md
  templates/
    audit-report.md
    refactor-plan.md
  schemas/
    findings.schema.json
    violations.schema.json
    changes.schema.json

agents/
  scout.md
  principles-auditor.md
  refactor-executor.md
  doc-updater.md
```

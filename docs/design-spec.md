# Design specification — general-purpose quality-team

> This document is the specification the skill was built from, kept as a record
> of intent. It describes what the pipeline **must** do;
> [`architecture.md`](architecture.md) describes what it actually does.
> Where the two diverge, the implementation in
> [`quality-team/`](../quality-team/) is authoritative.

Build and maintain a `quality-team` skill usable on any code project. The core
of the skill must not assume React, Tauri, TypeScript, Rust, npm, Biome, Clippy
or any other specific tool.

## Target architecture

Agent chain:

```text
scout -> principles-auditor -> refactor-executor -> doc-updater
```

The orchestrator skill stays a router:
- parses `scope` and `mode`
- detects the project profile
- loads the universal references
- loads the optional playbooks only when they apply
- spawns the agents
- presents a plan before any modification
- summarises the final result

## General principle

The pipeline must work even when no specialised tool is available. Qartez,
Lizard, linters, typecheckers and test runners enrich the analysis, but none of
them is required globally.

If no validation is detected:
- the audit carries on
- the plan states `validation: skipped`
- automatic refactoring stays more conservative

## Main files

```text
quality-team/
  SKILL.md
  README.md
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

## Universal references

`references/principles.md` holds the rules applicable to any codebase:
responsibility, source of truth, contracts, invariants, errors, side effects,
duplication, naming, complexity and living documentation.

`references/safe-refactor.md` defines only the safe, universal refactorings:
removing confirmed dead code, debug logs, commented-out blocks, local constants,
confirmed renames, small local extractions and documentation.

`references/ai-smells.md` describes the patterns of AI-generated code, without
depending on any framework.

## Optional playbooks

Playbooks are stack-specific extensions. They are loaded only when the detected
project matches:
- `react-ts.md`: React / TypeScript / Tauri
- `rust.md`: Rust

A playbook that does not apply must never produce a violation.

## JSON outputs

`findings.json` holds:
- scope
- detected project profile
- tools used
- candidate validation commands
- hotspots, dead code, complexity, clones, lint, unused dependencies, errors

`violations.json` holds:
- blocking, important, nit, suggestion
- ai_smells
- manual_verify

`changes.json` holds:
- applied
- skipped
- validation_results, in a generic form:

```json
[
  {
    "name": "test",
    "command": "npm test",
    "status": "pass",
    "output_path": ".claude/quality-team/test.out"
  }
]
```

## Validation

Validation is auto-detected:
- scripts declared by the project: `test`, `lint`, `typecheck`, `check`, `build`
- native commands when the manifest/config is present
- otherwise `skipped`

The refactor-executor uses only the commands detected in
`.claude/quality-team/validation_commands.json`.

## Asking the user

Before any refactoring, the skill generates
`.claude/quality-team/refactor_plan.md` from `templates/refactor-plan.md`, shows
it to the user, and waits for an explicit approval.

Without an explicit approval, no code is modified and the run continues as a
report only.

## Documentation

`README.md`, `INSTALL.md` and `MCP_CHECKLIST.md` must present the skill as
general-purpose. Stack-specific tools must always be described as optional,
depending on the target project.

# Architecture — contracts and security model

This document details how the pipeline works internally. For installation and
day-to-day usage, see the [README](../README.md).

## Why file-based contracts

The four agents do not hand information to each other in conversation. Each one
writes a JSON artifact to disk, which the next one reads back. That choice has
three direct consequences:

- **A failing step is visible.** The orchestrator checks that the output file
  exists after each phase and stops with an explicit message when it is missing,
  instead of letting the next agent work on nothing.
- **Every run is inspectable.** The artifacts stay in `.claude/quality-team/`
  after the run. A disagreement about a violation is settled by opening
  `violations.json`, not by re-running the analysis.
- **Agents are replaceable.** Any agent that honours the input and output schema
  can be substituted for the one shipped here.

The schemas live in [`quality-team/schemas/`](../quality-team/schemas/).

## The three contracts

### `findings.json` — scout → principles-auditor

Measured facts, with no qualitative judgment.

| Field | Type | Content |
|---|---|---|
| `scope` | string | Path analysed |
| `generated_at` | string | ISO timestamp |
| `project` | object | `manifests`, `languages`, `frameworks`, `playbooks_applicable`, `validation_commands` |
| `tools_used` | string[] | Tools actually executed |
| `validation_candidates` | validationCommand[] | Detected commands |
| `hotspots` | object[] | `file`, `score`, `reason`, `blast_radius` — sorted by descending score, 20 maximum |
| `dead_code` | object[] | `file`, `symbol`, `type`, `tool` — only when confirmed by a cross-file tool |
| `complexity` | object[] | `file`, `fn`, `ccn`, `nloc`, `params`, `threshold_breached` |
| `clones` | object[] | `files`, `lines`, `tool` |
| `lint` | object[] | `file`, `line`, `rule`, `severity` (`error` \| `warn` \| `info`), `message`, `tool` |
| `unused_deps` | object[] | `package`, `manager`, `type`, `tool` |
| `errors` | object[] | `tool`, `error` — missing tools, timeouts, parse failures |

The `errors` field is structural: an unavailable tool does not interrupt the
scout, it produces an entry and the analysis carries on. That is what lets the
pipeline run with no external tooling at all.

`$defs.validationCommand`: `name`, `command`, `reason`, `applies_to`
(the first three are required).

### `violations.json` — principles-auditor → refactor-executor

Classified judgments, each tied to a principle and to a piece of evidence.

| Field | Type | Content |
|---|---|---|
| `generated_at` | string | ISO timestamp |
| `files_analyzed` | integer | Number of files actually read |
| `blocking` | violation[] | Probable bug, data loss, silent error, security, broken external contract |
| `important` | violation[] | Significant debt, high complexity, structural duplication |
| `nit` | violation[] | Style, minor naming, simple documentation |
| `suggestion` | violation[] | Optional or uncertain improvement |
| `ai_smells` | object[] | `file`, `rule`, `description`, `severity` |
| `manual_verify` | object[] | `file`, `reason`, `recommendation` |

`$defs.violation`: `file`, `line`, `principle`, `description`, `evidence`,
`fix_hint` — `file`, `principle`, `description` and `fix_hint` are required.
A violation with no suggested fix is therefore not representable.

Two rules govern the classification:

- **Uncertainty is ranked downward.** When in doubt, the auditor classifies as
  `suggestion`, never as `blocking`.
- **`manual_verify` is exclusive.** A file placed in `manual_verify` cannot
  appear at the same time as an automatic candidate under `blocking` or
  `important`. That is what stops the executor from touching a file the auditor
  judged too risky.

### `changes.json` — refactor-executor → doc-updater

What was done, what was not, and why.

| Field | Type | Content |
|---|---|---|
| `generated_at` | string | ISO timestamp |
| `applied` | object[] | `file`, `type`, `description`, `blast_radius_checked`, `validated`, `tools_passed` — all required |
| `skipped` | object[] | `file`, `reason` required; `validation_output`, `recommendation`, `blast_radius` optional |
| `validation_results` | validationResult[] | `name`, `command`, `status` (`pass` \| `fail` \| `skipped`), `output_path`, `summary` |

Because `validated` and `tools_passed` are required on every `applied` entry, a
change applied with no validation available is recorded as exactly that
(`validated: false`, `tools_passed: []`) rather than presented as verified.

## Run sequence

| Phase | Actor | Input | Output |
|---|---|---|---|
| **0** — Detection | orchestrator | manifests, extensions in the scope | `project_profile.json`, `validation_commands.json`, `baseline_validation.json` |
| **0b** — References | orchestrator | `references/`, `playbooks/` | enriched prompts |
| **1** — Mapping | `scout` | scope, project profile | `findings.json` |
| **2** — Audit | `principles-auditor` | `findings.json`, references | `violations.json` |
| **2b** — Plan | orchestrator | `findings`, `violations`, `validation_commands` | `refactor_plan.md` + **user approval** |
| **3** — Execution | `refactor-executor` | approved plan | `changes.json` |
| **4** — Report | `doc-updater` | every artifact | `REFACTOR_REPORT.md` |
| **5** — Summary | orchestrator | `baseline_validation.json` vs post-refactor | comparison displayed |

Phase 0 records a **validation baseline** before any modification. Phase 5
replays the same commands and displays the before → after comparison. Without
that baseline, a test that was already red before the run would be blamed on the
refactoring.

Phase 3 is skipped when `mode = audit-only`, or when the plan did not receive an
explicit approval. In `audit-only` mode, the plan is presented as a
recommendation.

## Security model

### Per-agent permissions

The restrictions are declared in each agent's frontmatter, so they are enforced
by Claude Code rather than by a textual instruction.

| Agent | `tools` | Writes possible |
|---|---|---|
| `scout` | `Read`, `Glob`, `Grep`, `Bash` | none |
| `principles-auditor` | `Read`, `Grep` | none — not even `Bash` |
| `refactor-executor` | `Read`, `Write`, `Edit`, `MultiEdit`, `Bash` | source code, within the perimeter of the plan |
| `doc-updater` | `Read`, `Write`, `Edit` | documentation only, no `Bash` |

The orchestrator skill itself is limited to `Read` and `Bash`: it cannot modify
a file directly.

### Risk scale

[`references/safe-refactor.md`](../quality-team/references/safe-refactor.md)
splits the operations across four levels:

1. **Always safe once the tooling confirms it** — removing confirmed dead code,
   non-functional debug logs, commented-out blocks longer than 3 lines;
   extracting a named constant within the same file; fixing public documentation
   without touching a signature or the logic.
2. **Safe with post-change validation** — renaming with an update of every
   confirmed usage, extracting a small local function, moving an internal
   function, simplifying a condition. If no validation is available, these
   operations are still proposed in the plan but treated as higher risk.
3. **Never without separate human review** — public signature or return type,
   deleting a whole file, authentication / authorisation / encryption / secrets
   / sessions, migrations and database schemas, tests beyond an import required
   by a rename, build or CI configuration, a new dependency, an external API.
4. **Never touched** — generated or vendored files, build directories
   (`dist/`, `build/`, `target/`, `.next/`, `out/`, `.cache/`, `vendor/`),
   lockfiles, files marked `DO NOT EDIT` or `GENERATED`, and every file listed
   in `violations.manual_verify`.

### Per-file protocol

The executor applies the same sequence to every file, and stops at the first
blocking condition:

1. Check the blacklist and `manual_verify`.
2. Check the blast radius through Qartez if the file is a hotspot.
3. Read the file, identify the smallest sufficient change.
4. Apply only what appears in `refactor_plan.md`.
5. Run the applicable validations.
6. **If a validation fails, revert the single file touched** and log the skip.
7. Move on to the next file.

The revert is deliberately file-grained: one failure does not undo the changes
already validated on the other files.

### Normalised skip reasons

`manual-verify` · `blacklisted` · `blast-radius-too-high` ·
`not-in-approved-plan` · `missing-safe-refactor-rules` · `validation-unavailable` ·
`reverted:validation-failed`

The `missing-safe-refactor-rules` case is worth noting: if the safety rules did
not reach the executor, it applies **no** change at all and puts every candidate
in `skipped`. A missing guardrail is not treated as an absence of constraint.

## Extending the pipeline

### Adding a playbook

A playbook is a file in `quality-team/playbooks/`, loaded conditionally. Three
hook points:

1. Create `quality-team/playbooks/<stack>.md`, cross-referencing the rules with
   the universal principles (`P1`–`P10`) rather than redefining them.
2. Declare the loading condition in phase 0b of
   [`SKILL.md`](../quality-team/SKILL.md).
3. Make sure the scout records the stack in
   `findings.project.playbooks_applicable` — the auditor loads only the
   playbooks listed there.

The constraint to respect: **a playbook that does not apply must produce no
violation.** That is what guarantees that adding a playbook does not degrade the
analysis of the projects it has nothing to do with.

### Adding a universal rule

Rules that apply to any language belong in
[`references/principles.md`](../quality-team/references/principles.md)
(structural principles) or
[`references/ai-smells.md`](../quality-team/references/ai-smells.md) (patterns of
generated code). No change to `SKILL.md` is needed: both files are always
loaded.

A rule only becomes automatically actionable if it is also covered by
`safe-refactor.md`. Otherwise it produces findings that the executor will leave
in `skipped`, with the reason `not-in-approved-plan`.

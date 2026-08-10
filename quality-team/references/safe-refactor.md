# Safe refactor rules
# Used by: refactor-executor
# Universal rules, independent of language and framework.
---

Automatic refactoring must be conservative. Playbooks may add specific cases,
but only for a detected stack and only when the plan lists them explicitly.

## Always safe once the tooling confirms it

These changes stay local and must not alter public behaviour:

- Remove dead code confirmed by a cross-file analysis or by Qartez.
- Remove non-functional debug logs, excluding audit/security/observability logs.
- Remove commented-out blocks longer than 3 lines.
- Extract a named constant within the same file.
- Fix public documentation without changing a signature or the logic.

## Safe with post-change validation

These operations require the validations detected in
`.claude/quality-team/validation_commands.json`:

- Rename a symbol and update every confirmed usage.
- Extract a small local function without changing the public interface.
- Move an internal function to a neighbouring module, updating every import.
- Replace a magic value with a constant already introduced locally.
- Simplify a condition or a guard without changing the cases it covers.

If no project validation is available, these changes may still be proposed in
the plan, but must be treated as higher risk.

## Never without separate human review

- Change a public signature.
- Change a public return type or format.
- Delete a whole file.
- Touch authentication, authorisation, encryption, secrets, sessions or permissions.
- Touch migrations, database schemas or persisted formats.
- Change tests beyond an import/path required by a rename.
- Change the build, packaging, CI or deployment configuration.
- Introduce a new dependency.
- Change an external API or a network protocol.

## Never touch

- Generated or vendored files.
- Build/cache directories (`dist/`, `build/`, `target/`, `.next/`, `out/`,
  `.cache/`, `vendor/`, and equivalents).
- Lockfiles.
- Files marked `DO NOT EDIT` or `GENERATED`.
- Files listed in `violations.manual_verify`.

## Per-file protocol

1. Check the blacklist and `manual_verify`.
2. Check the blast radius with Qartez for hotspots, when available.
3. Read the file and identify the smallest sufficient change.
4. Apply only what is listed in `refactor_plan.md`.
5. Run the applicable detected validations.
6. If a validation fails, revert only the file you touched and log the skip.
7. Move on to the next file.

## Standard skip reasons

- `manual-verify`
- `blacklisted`
- `blast-radius-too-high`
- `not-in-approved-plan`
- `missing-safe-refactor-rules`
- `validation-unavailable`
- `reverted:validation-failed`

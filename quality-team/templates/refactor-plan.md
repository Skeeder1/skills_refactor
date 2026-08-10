# quality-team action plan

## Scope

- Scope analysed: {{scope}}
- Requested mode: {{mode}}
- Languages detected: {{languages}}
- Playbooks loaded: {{playbooks}}

## Planned validations

| Name | Command | Baseline status | Reason |
|------|---------|-----------------|--------|
| {{validation_name}} | `{{validation_command}}` | {{baseline_status}} | {{reason}} |

If no command is detected, state: automatic validation unavailable.

## Proposed changes

| File | Severity | Problem | Planned change | Risk | Validation |
|------|----------|---------|----------------|------|------------|
| {{file}} | {{severity}} | {{problem}} | {{planned_change}} | {{risk}} | {{validation}} |

## Files left untouched

| File | Reason |
|------|--------|
| {{file}} | {{reason}} |

## Limits

- Changes outside the scope are excluded.
- Generated, vendored, lockfile and `DO NOT EDIT` files are excluded.
- `manual_verify` files are excluded until a separate human review.
- No refactoring starts without explicit approval from the user.

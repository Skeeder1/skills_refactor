# Installation guide — quality-team

## Minimum requirements

- Claude Code with support for skills and agents.
- A source project to analyse.
- Git recommended, so that diffs and reverts stay safe.

No language, runtime or package manager is required globally. Validation tools
are detected inside each target project.

## Recommended tooling

| Tool | Role | Status |
|------|------|--------|
| Qartez MCP | structure, hotspots, blast radius, dead code, AST clones | recommended |
| Lizard | multi-language complexity | recommended |
| The project's own linters/typecheckers | validation after refactoring | optional, depends on the stack |

Examples of stack-specific tools detected when present: npm/pnpm/yarn scripts,
Cargo, Go, pytest/ruff/mypy, Maven, Gradle, Make, Composer, Bundler, dotnet.

## Get the repository

```bash
git clone https://github.com/Skeeder1/skills_refactor.git
cd skills_refactor
```

The commands below are run from the root of the cloned repository.

## Install the skill and the agents

### Linux / macOS

```bash
mkdir -p ~/.claude/skills ~/.claude/agents
cp -r quality-team ~/.claude/skills/quality-team
cp agents/*.md ~/.claude/agents/
```

### Windows (PowerShell)

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills", "$env:USERPROFILE\.claude\agents"
Copy-Item -Recurse -Force ".\quality-team" "$env:USERPROFILE\.claude\skills\quality-team"
Copy-Item ".\agents\*.md" "$env:USERPROFILE\.claude\agents\"
```

To reinstall over an existing copy on Linux/macOS, delete
`~/.claude/skills/quality-team` first — otherwise `cp -r` nests the directory
inside itself instead of replacing it.

## References and playbooks

The references ship inside `quality-team/` and are injected by the
orchestrator. Playbooks are optional and loaded only when the target project
matches the detected stack.

## Optional MCP servers

See `MCP_CHECKLIST.md` to configure:
- Qartez MCP, recommended for every project
- @knip/mcp, useful for JavaScript/TypeScript projects
- SonarQube MCP, useful in enterprise environments

## Usage

In Claude Code, from the root of the target project:

```text
/quality-team .
/quality-team src audit-only
/quality-team packages/api refactor
```

Expected outputs:
- `.claude/quality-team/findings.json`
- `.claude/quality-team/violations.json`
- `.claude/quality-team/refactor_plan.md`
- `.claude/quality-team/changes.json` if the refactoring was approved
- `REFACTOR_REPORT.md`

## Quick verification

### Linux / macOS

```bash
test -f ~/.claude/skills/quality-team/SKILL.md && echo "skill installed"
ls ~/.claude/agents/
```

The four expected files in `~/.claude/agents/` are `scout.md`,
`principles-auditor.md`, `refactor-executor.md` and `doc-updater.md`.

### Windows (PowerShell)

```powershell
Test-Path "$env:USERPROFILE\.claude\skills\quality-team\SKILL.md"
Get-ChildItem "$env:USERPROFILE\.claude\agents"
```

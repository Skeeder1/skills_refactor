# MCP and tooling checklist — quality-team

The skill works without any MCP, but some tools improve precision
considerably. No stack-specific tool is mandatory for every project.

## Recommended MCP servers

### Qartez MCP — recommended for every project

**Why:**
- Structural map of the codebase
- Hotspots: complexity × coupling × churn
- Blast radius before a modification
- Cross-file dead code
- AST clones

**Install, Windows PowerShell:**

```powershell
powershell -ExecutionPolicy Bypass -c "iwr https://raw.githubusercontent.com/kuberstar/qartez-mcp/main/install.ps1 -useb | iex"
```

**Config:**

```json
{
  "mcpServers": {
    "qartez": {
      "command": "qartez",
      "args": []
    }
  }
}
```

**Agents:**
- scout: `qartez_map`, `qartez_hotspots`, `qartez_unused`, `qartez_clones`, `qartez_deps`
- principles-auditor: `qartez_read`, `qartez_outline`, `qartez_refs`, `qartez_calls`
- refactor-executor: `qartez_impact`, `qartez_refs`

### @knip/mcp — optional, JavaScript/TypeScript

**Why:**
- JS/TS dead code
- Unused exports
- Unused dependencies
- Better handling of barrel files and framework entry points

**Config:**

```json
{
  "mcpServers": {
    "knip": {
      "command": "npx",
      "args": ["@knip/mcp"]
    }
  }
}
```

### SonarQube MCP — optional, enterprise

Useful if your organisation already uses SonarQube for smells, security,
duplication, quality gates and history.

## Recommended CLIs, per project

| Project | Examples of detected commands |
|--------|---------------------------------|
| Multi-language | `lizard` |
| JS/TS | `npm/pnpm/yarn/bun test\|lint\|typecheck\|build` scripts, Biome, ESLint, TypeScript, Knip |
| Rust | `cargo test`, `cargo check`, `cargo clippy` |
| Python | `pytest`, `ruff`, `mypy`, `python -m compileall` |
| Go | `go test ./...`, `go vet ./...` |
| Java/Kotlin | Maven or Gradle `test/check` |
| PHP | Composer scripts, PHPUnit, PHPStan/Psalm |
| Ruby | Bundler/RSpec/RuboCop |
| .NET | `dotnet test`, `dotnet build` |

The skill prefers the commands declared by the project. If nothing is detected,
it carries on with the audit and marks the validation as `skipped`.

# Checklist MCP et outils — quality-team

Le skill fonctionne sans MCP, mais certains outils améliorent fortement la
précision. Aucun outil stack-spécifique n'est obligatoire pour tous les projets.

## MCP recommandés

### Qartez MCP — recommandé pour tous les projets

**Pourquoi :**
- Cartographie structurelle du codebase
- Hotspots complexité × couplage × churn
- Blast radius avant modification
- Dead code cross-fichiers
- Clones AST

**Install Windows PowerShell :**

```powershell
powershell -ExecutionPolicy Bypass -c "iwr https://raw.githubusercontent.com/kuberstar/qartez-mcp/main/install.ps1 -useb | iex"
```

**Config :**

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

**Agents :**
- scout : `qartez_map`, `qartez_hotspots`, `qartez_unused`, `qartez_clones`, `qartez_deps`
- principles-auditor : `qartez_read`, `qartez_outline`, `qartez_refs`, `qartez_calls`
- refactor-executor : `qartez_impact`, `qartez_refs`

### @knip/mcp — optionnel JavaScript/TypeScript

**Pourquoi :**
- Dead code JS/TS
- Exports inutilisés
- Dépendances inutilisées
- Meilleure gestion des barrel files et entry points framework

**Config :**

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

### SonarQube MCP — optionnel enterprise

Utile si ton organisation utilise déjà SonarQube pour smells, sécurité,
duplication, quality gates et historique.

## CLI recommandées selon projet

| Projet | Exemples de commandes détectées |
|--------|---------------------------------|
| Multi-langage | `lizard` |
| JS/TS | scripts `npm/pnpm/yarn/bun test|lint|typecheck|build`, Biome, ESLint, TypeScript, Knip |
| Rust | `cargo test`, `cargo check`, `cargo clippy` |
| Python | `pytest`, `ruff`, `mypy`, `python -m compileall` |
| Go | `go test ./...`, `go vet ./...` |
| Java/Kotlin | Maven ou Gradle `test/check` |
| PHP | Composer scripts, PHPUnit, PHPStan/Psalm |
| Ruby | Bundler/RSpec/RuboCop |
| .NET | `dotnet test`, `dotnet build` |

Le skill préfère les commandes déclarées par le projet. Si rien n'est détecté,
il continue l'audit et marque la validation comme `skipped`.

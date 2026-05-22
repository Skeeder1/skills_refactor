# Checklist MCP — quality-refactor-team

Ces serveurs MCP améliorent l'analyse mais ne sont **pas obligatoires**.
La chaîne d'agents fonctionne sans eux via les CLI déterministes (knip, lizard, biome, clippy).

---

## MCP servers à ajouter dans `~/.claude.json` ou le settings.json de ton projet

### 1. Qartez MCP (intelligence structurelle — recommandé)

**Pourquoi :**
- Hotspots (complexité × couplage × churn) — identifie les fichiers les plus risqués
- Blast radius avant édition — évite de casser des dépendances cachées
- Dead code cross-fichiers via PageRank — moins de faux positifs que knip seul
- Clones structurels via AST hashing — duplication non détectable par regex
- Réduit la consommation de tokens de ~94% par rapport à lire les fichiers bruts

**Install (Windows PowerShell) :**
```powershell
powershell -ExecutionPolicy Bypass -c "iwr https://raw.githubusercontent.com/kuberstar/qartez-mcp/main/install.ps1 -useb | iex"
```

**Config settings.json :**
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

**Outils utilisés par les agents :**

| Agent | Outils Qartez utilisés |
|-------|----------------------|
| scout | `qartez_map`, `qartez_hotspots`, `qartez_unused`, `qartez_clones`, `qartez_deps` |
| principles-auditor | `qartez_read`, `qartez_outline`, `qartez_refs`, `qartez_calls` |
| refactor-executor | `qartez_impact` (OBLIGATOIRE avant tout fichier hotspot score > 5), `qartez_refs` |

---

### 2. @knip/mcp (dead code TypeScript — recommandé pour les projets TS/JS)

**Pourquoi :**
- Détecte les exports/fichiers/dépendances inutilisés en cross-fichiers
- Comprend les barrel files (`index.ts` avec `export *`) et évite les faux positifs
- Identifie les framework entry points (Next.js pages, Vite plugins, etc.) que knip CLI peut rater
- Complète Qartez pour le dead code TypeScript

**Install :**
```bash
npm install -D knip
```

**Config settings.json :**
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

**Agents qui l'utilisent :**
- `scout` : enrichit `knip_raw.json` avec une analyse plus fine des barrel files
- `principles-auditor` : vérifie les dead exports avant de les marquer `blocking`

---

### 3. SonarQube MCP (optionnel — qualité enterprise)

**Pourquoi :**
- Code smells enterprise, duplication, analyse de sécurité (OWASP), quality gates
- Gratuit en Community Build self-hosted
- Historique des métriques dans le temps

**Install :**
Docker self-hosted + MCP server officiel SonarSource.
Voir : https://docs.sonarsource.com/sonarqube/latest/

**Agents qui l'utilisent :**
- `scout` : source additionnelle pour les hotspots et violations de sécurité

---

## CLI tools à installer globalement

Ces outils sont appelés directement par les agents via Bash — ils doivent être dans le `PATH`.
Ils sont **obligatoires** (la chaîne en dépend même sans MCP).

| Outil | Install | Version min | Utilisé par |
|-------|---------|-------------|------------|
| **Biome** | `npm install -g @biomejs/biome` | 1.8+ | scout, refactor-executor |
| **Knip** | `npm install -D knip` (par projet) | 5.0+ | scout |
| **Lizard** | `pip install lizard` | 1.17+ | scout (complexité cyclomatique) |
| **TypeScript** | `npm install -D typescript` (par projet) | 5.0+ | refactor-executor |
| **Cargo/Clippy** | Via rustup (inclus) | stable | scout, refactor-executor |

---

## Vérification rapide

Lance ce bloc pour vérifier que les CLI requis sont disponibles :

```bash
# Vérification des outils obligatoires
Write-Output "=== Vérification CLI quality-team ==="
@(
  @{ Name = "biome";   Cmd = "npx biome --version" },
  @{ Name = "knip";    Cmd = "npx knip --version" },
  @{ Name = "lizard";  Cmd = "lizard --version" },
  @{ Name = "tsc";     Cmd = "npx tsc --version" },
  @{ Name = "cargo";   Cmd = "cargo --version" },
  @{ Name = "qartez";  Cmd = "qartez --version" }
) | ForEach-Object {
  try {
    $version = Invoke-Expression $_.Cmd 2>$null
    Write-Output "  ✅ $($_.Name) : $version"
  } catch {
    Write-Output "  ❌ $($_.Name) : NON INSTALLÉ"
  }
}
```

---

## Priorité d'installation

1. **Biome** — utilisé par tous les agents, indispensable
2. **Knip + Lizard** — utilisés par scout, sans eux findings.json sera partiel
3. **Qartez MCP** — améliore significativement la précision (blast radius, clones AST)
4. **@knip/mcp** — réduit les faux positifs sur les projets TypeScript avec barrel files
5. **SonarQube** — optionnel, pour les projets enterprise avec historique de métriques

# Guide d'installation — quality-refactor-team

## Prérequis

Avant d'installer, vérifie que ces outils sont disponibles :

| Outil | Version min | Vérification |
|-------|-------------|-------------|
| Node.js | 20+ | `node --version` |
| npm | 10+ | `npm --version` |
| Python | 3.8+ | `python --version` |
| Rust + Cargo | stable | `cargo --version` _(optionnel, requis si tu analyses des projets Tauri)_ |
| pip (lizard) | — | `pip install lizard && lizard --version` |

---

## Étape 1 — Installer les CLI requis

```bash
# Biome (lint JS/TS)
npm install -g @biomejs/biome

# Lizard (complexité cyclomatique)
pip install lizard

# Knip (dead code TypeScript) — à installer dans chaque projet analysé
npm install -D knip

# TypeScript — à installer dans chaque projet analysé
npm install -D typescript
```

---

## Étape 2 — Installer le skill orchestrateur

Copie le dossier `quality-team/` dans le répertoire des skills Claude Code :

```powershell
# Windows
Copy-Item -Recurse -Force "C:\Projets\skills-refactor\quality-team" "$env:USERPROFILE\.claude\skills\quality-team"

# Vérification
Get-ChildItem "$env:USERPROFILE\.claude\skills\quality-team"
# Doit afficher : SKILL.md, README.md, schemas/
```

---

## Étape 3 — Installer les agents

Copie les fichiers d'agents dans le répertoire des agents Claude Code :

```powershell
# Windows — créer le dossier si absent
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\agents"

# Copier les 4 agents
Copy-Item "C:\Projets\skills-refactor\agents\*.md" "$env:USERPROFILE\.claude\agents\"

# Vérification
Get-ChildItem "$env:USERPROFILE\.claude\agents"
# Doit afficher : scout.md, principles-auditor.md, refactor-executor.md, doc-updater.md
```

---

## Étape 4 — Copier les références dans le projet à analyser

Les agents lisent `references/` et `playbooks/` depuis le contexte du projet analysé.
Copie-les dans le projet cible sous `.claude/` :

```powershell
# Dans le répertoire du projet à analyser
$project = "C:\chemin\vers\ton\projet"

New-Item -ItemType Directory -Force -Path "$project\.claude"

# Références obligatoires (lues par principles-auditor)
Copy-Item -Recurse -Force "C:\Projets\skills-refactor\references" "$project\.claude\references"

# Playbooks spécifiques au stack (choisir selon le projet)
Copy-Item -Recurse -Force "C:\Projets\skills-refactor\playbooks" "$project\.claude\playbooks"

# Vérification
Get-ChildItem "$project\.claude" -Recurse
```

> **Note :** Si le projet est analysé depuis son répertoire racine, les agents chercheront
> `references/` et `playbooks/` dans `.claude/`. Adapte si ta structure diffère.

---

## Étape 5 — Configurer les MCP (optionnel mais recommandé)

Consulte `MCP_CHECKLIST.md` pour les instructions d'installation de :
- **Qartez MCP** — hotspots, blast radius, dead code AST (recommandé)
- **@knip/mcp** — dead code TypeScript avec barrel files (recommandé pour TS/JS)

---

## Étape 6 — Utilisation

Dans Claude Code, depuis le répertoire racine du projet à analyser :

```
# Analyse complète + refactoring (mode par défaut)
/quality-team src/

# Audit uniquement (pas de modifications)
/quality-team . audit-only

# Refactoring d'un sous-répertoire
/quality-team src/components refactor

# Analyse à la racine du projet
/quality-team . all
```

**Output attendu :**
- `.claude/quality-team/findings.json` — résultats d'analyse statique
- `.claude/quality-team/violations.json` — violations classifiées
- `.claude/quality-team/changes.json` — journal des modifications
- `REFACTOR_REPORT.md` — rapport complet à la racine du projet

---

## Vérification de l'installation

```powershell
Write-Output "=== Vérification installation quality-team ==="

# Skill
$skillOk = Test-Path "$env:USERPROFILE\.claude\skills\quality-team\SKILL.md"
Write-Output "Skill SKILL.md : $(if ($skillOk) { '✅' } else { '❌ MANQUANT' })"

# Agents
foreach ($agent in @("scout", "principles-auditor", "refactor-executor", "doc-updater")) {
  $ok = Test-Path "$env:USERPROFILE\.claude\agents\$agent.md"
  Write-Output "Agent $agent.md : $(if ($ok) { '✅' } else { '❌ MANQUANT' })"
}

# CLI
foreach ($item in @(
  @{ Name = "biome";  Cmd = "npx biome --version" },
  @{ Name = "lizard"; Cmd = "lizard --version" },
  @{ Name = "tsc";    Cmd = "npx tsc --version" }
)) {
  try {
    $v = Invoke-Expression $item.Cmd 2>$null
    Write-Output "CLI $($item.Name) : ✅ ($v)"
  } catch {
    Write-Output "CLI $($item.Name) : ❌ NON INSTALLÉ"
  }
}
```

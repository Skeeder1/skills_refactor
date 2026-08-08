# Guide d'installation — quality-team

## Prérequis minimaux

- Claude Code avec support des skills et agents.
- Un projet source à analyser.
- Git recommandé pour permettre les diffs et reverts sûrs.

Aucun langage, runtime ou gestionnaire de paquets n'est obligatoire globalement.
Les outils de validation sont détectés dans chaque projet cible.

## Outils recommandés

| Outil | Rôle | Statut |
|-------|------|--------|
| Qartez MCP | structure, hotspots, blast radius, dead code, clones AST | recommandé |
| Lizard | complexité multi-langage | recommandé |
| Linters/typecheckers du projet | validation après refactor | optionnels selon stack |

Exemples d'outils stack-spécifiques détectés si présents : scripts npm/pnpm/yarn,
Cargo, Go, pytest/ruff/mypy, Maven, Gradle, Make, Composer, Bundler, dotnet.

## Récupérer le dépôt

```bash
git clone https://github.com/Skeeder1/skills_refactor.git
cd skills_refactor
```

Les commandes ci-dessous s'exécutent depuis la racine du dépôt cloné.

## Installer le skill et les agents

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

Pour réinstaller par-dessus une version existante sous Linux/macOS, supprimer
d'abord `~/.claude/skills/quality-team` : sinon `cp -r` imbrique le dossier au
lieu de le remplacer.

## Références et playbooks

Les références sont embarquées dans `quality-team/` et injectées par
l'orchestrateur. Les playbooks sont optionnels et chargés seulement si le projet
cible correspond au stack détecté.

## MCP optionnels

Consulte `MCP_CHECKLIST.md` pour configurer :
- Qartez MCP, recommandé pour tous les projets
- @knip/mcp, utile pour projets JavaScript/TypeScript
- SonarQube MCP, utile pour environnements enterprise

## Utilisation

Dans Claude Code, depuis la racine du projet cible :

```text
/quality-team .
/quality-team src audit-only
/quality-team packages/api refactor
```

Sorties attendues :
- `.claude/quality-team/findings.json`
- `.claude/quality-team/violations.json`
- `.claude/quality-team/refactor_plan.md`
- `.claude/quality-team/changes.json` si refactor validé
- `REFACTOR_REPORT.md`

## Vérification rapide

### Linux / macOS

```bash
test -f ~/.claude/skills/quality-team/SKILL.md && echo "skill installé"
ls ~/.claude/agents/
```

Les quatre fichiers attendus dans `~/.claude/agents/` sont `scout.md`,
`principles-auditor.md`, `refactor-executor.md` et `doc-updater.md`.

### Windows (PowerShell)

```powershell
Test-Path "$env:USERPROFILE\.claude\skills\quality-team\SKILL.md"
Get-ChildItem "$env:USERPROFILE\.claude\agents"
```

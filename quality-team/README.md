# quality-team — Chaîne d'agents pour l'audit et le refactoring automatique

`/quality-team` lance une séquence de 4 sous-agents spécialisés qui analysent,
nettoient, et documentent un codebase de façon autonome et reproductible.

---

## Ce que fait la chaîne

| Agent | Rôle | Output |
|-------|------|--------|
| **scout** | Cartographie objective via CLI déterministes (knip, lizard, biome, clippy) et Qartez MCP optionnel. Read-only strict. | `findings.json` |
| **principles-auditor** | Applique 10 principes clean code + 27 AI smells sur les fichiers flaggés par scout. Classifie par sévérité. Read-only strict. | `violations.json` |
| **refactor-executor** | Applique les changements sûrs (blocking + important) avec validation tsc/biome/clippy après chaque fichier. Revert si échec. | `changes.json` |
| **doc-updater** | Met à jour le JSDoc des fonctions modifiées, AGENTS.md si des modules ont bougé, génère REFACTOR_REPORT.md. Ne touche jamais le code source. | `REFACTOR_REPORT.md` |

---

## Structure des fichiers produits

```
.claude/quality-team/
  findings.json          ← analyse statique (hotspots, dead code, complexité, lint)
  violations.json        ← violations qualitatives classifiées (blocking/important/nit/...)
  changes.json           ← journal des modifications (absent en mode audit-only)
  baseline_tsc.txt       ← état tsc avant refactoring
  baseline_biome.json    ← état biome avant refactoring
  baseline_clippy.json   ← état clippy avant refactoring (si Rust)
  post_tsc.txt           ← état tsc après refactoring
  post_biome.json        ← état biome après refactoring

REFACTOR_REPORT.md       ← rapport lisible à la racine du projet analysé
```

---

## Commandes d'utilisation

```
# Mode complet : audit + refactoring + rapport (défaut)
/quality-team src/

# Mode audit uniquement (aucune modification de fichier)
/quality-team src/ audit-only

# Mode refactoring explicite sur un sous-répertoire
/quality-team src/components refactor
```

**Output attendu (mode complet) :**
```
✅ Scout terminé — findings.json généré
✅ Audit terminé :
   - Blocking : 3
   - Important : 8
   - AI smells : 5
   - Manual verify : 2
✅ Refactor terminé :
   - Changements appliqués : 9
   - Fichiers skippés : 2
═══════════════════════════════════════
  QUALITY-TEAM — RAPPORT FINAL
  TypeScript  : fail → pass  [✅ amélioré]
  Biome       : 12 → 3       [✅ amélioré]
  Rapport complet : REFACTOR_REPORT.md
═══════════════════════════════════════
```

---

## Modes disponibles

| Mode | Comportement | Quand l'utiliser |
|------|-------------|-----------------|
| `audit-only` | Scout + auditor + doc-updater. Aucune modification de code. Génère findings.json + violations.json + rapport. | Première analyse, avant une release, pour avoir une vue sans risque. |
| `refactor` | Chaîne complète avec exécution des corrections sûres. | Pour appliquer les corrections après audit. |
| `all` (défaut) | Chaîne complète de scout à doc-updater. | Usage normal. |

---

## Limitations connues

- **Fichiers sans couverture de tests** → automatiquement placés dans `manual_verify`. L'exécuteur ne les touchera pas. Ajouter des tests, puis relancer `/quality-team`.

- **Faux positifs knip sur barrel files** → Si ton projet utilise des `index.ts` avec `export *`, installez `@knip/mcp` (voir `MCP_CHECKLIST.md`) pour réduire les faux positifs.

- **Qartez optionnel** → La chaîne fonctionne sans Qartez MCP, mais la précision des hotspots et du blast radius sera moindre. Certaines vérifications de dead code cross-fichiers seront moins fiables.

- **Baseline tsc déjà cassée** → Si `tsc --noEmit` échoue avant le refactoring, l'exécuteur passera quand même mais les validations post-modification seront moins fiables. Corriger la baseline d'abord.

- **Projets sans package.json ni Cargo.toml** → Le scout continuera avec les outils disponibles et loggera les erreurs dans `findings.errors`.

---

## Pour améliorer l'analyse

Voir `MCP_CHECKLIST.md` pour configurer :
- **Qartez MCP** — hotspots précis, blast radius, dead code AST
- **@knip/mcp** — moins de faux positifs TypeScript

Les règles et playbooks sont embarqués dans `quality-team/references/` et
`quality-team/playbooks/`. L'orchestrateur les injecte inline dans les prompts :
les sous-agents ne dépendent pas de fichiers `references/` à la racine du projet cible.

---

## Architecture

```
quality-team/
  SKILL.md          ← tu es ici — orchestrateur principal
  README.md         ← ce fichier
  references/       ← règles injectées inline par l'orchestrateur
    principles.md           ← 10 principes fondamentaux (P1-P10)
    clean-code-rules.md     ← Clean Code + Clean Architecture + WELC
    refactoring-rules.md    ← Refactoring (Fowler)
    ai-smells.md            ← 27 patterns AI-générés
    safe-refactor.md        ← règles de sécurité pour refactor-executor
  playbooks/        ← règles stack-spécifiques injectées si pertinentes
    react-ts.md             ← React + TypeScript + Tauri
    rust.md                 ← Rust + Tauri backend
  templates/
    audit-report.md         ← template REFACTOR_REPORT.md
  schemas/
    findings.schema.json    ← JSON Schema 2020-12 pour findings.json
    violations.schema.json  ← JSON Schema 2020-12 pour violations.json
    changes.schema.json     ← JSON Schema 2020-12 pour changes.json

agents/             ← sous-agents spawned par l'orchestrateur
  scout.md
  principles-auditor.md
  refactor-executor.md
  doc-updater.md
```

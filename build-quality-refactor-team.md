# Mission : construire quality-refactor-team dans C:\Projets\skills-refactor

Tu es Claude Code. Tu travailles de façon entièrement autonome.
Lis chaque phase complètement avant d'agir. Ne t'arrête que si une dépendance
réseau est inaccessible — dans ce cas, note le problème et continue avec ce
que tu as.

---

## Contexte et architecture

Tu vas construire une chaîne de 4 sous-agents pour analyser, nettoyer et
documenter un codebase automatiquement. L'orchestrateur spawn les agents
séquentiellement en passant un fichier JSON entre chaque étape.

Chaîne : `scout` → `principles-auditor` → `refactor-executor` → `doc-updater`

Les serveurs MCP (Qartez, @knip/mcp) sont disponibles en option aux 3 premiers
agents — ils permettent d'aller plus vite mais ne sont pas obligatoires. L'agent
fonctionne même s'ils ne sont pas installés. NE configure pas les MCP toi-même :
génère une checklist que l'utilisateur configurera manuellement.

---

## Phase 0 — Initialisation du projet

```bash
# Windows PowerShell
New-Item -ItemType Directory -Force -Path "C:\Projets\skills-refactor"
New-Item -ItemType Directory -Force -Path "C:\Projets\skills-refactor\quality-team"
New-Item -ItemType Directory -Force -Path "C:\Projets\skills-refactor\quality-team\schemas"
New-Item -ItemType Directory -Force -Path "C:\Projets\skills-refactor\agents"
New-Item -ItemType Directory -Force -Path "C:\Projets\skills-refactor\references"
New-Item -ItemType Directory -Force -Path "C:\Projets\skills-refactor\playbooks"
New-Item -ItemType Directory -Force -Path "C:\Projets\skills-refactor\templates"
```

Structure cible :

```
C:\Projets\skills-refactor\
  quality-team\
    SKILL.md                 ← orchestrateur principal
    README.md
    schemas\
      findings.schema.json
      violations.schema.json
      changes.schema.json
  agents\
    scout.md
    principles-auditor.md
    refactor-executor.md
    doc-updater.md
  references\
    clean-code-rules.md      ← récupéré depuis ciembor/agent-rules-books
    refactoring-rules.md     ← récupéré depuis ciembor/agent-rules-books
    ai-smells.md             ← construit depuis sober-coding patterns
    safe-refactor.md         ← règles de sécurité pour l'exécuteur
    principles.md            ← 10 principes clean code
  playbooks\
    react-ts.md              ← règles spécifiques React + TypeScript + Tauri
    rust.md                  ← règles spécifiques Rust + Cargo
  templates\
    audit-report.md          ← template REFACTOR_REPORT.md
  MCP_CHECKLIST.md           ← checklist pour config manuelle utilisateur
  INSTALL.md                 ← guide d'installation
```

---

## Phase 1 — Récupération des ressources communautaires

### 1a. Récupérer clean-code.mini.md depuis ciembor/agent-rules-books

Fetch cette URL et sauvegarde le contenu brut :
`https://raw.githubusercontent.com/ciembor/agent-rules-books/main/clean-code/clean-code.mini.md`

Si la réponse est 200, enregistre dans `C:\Projets\skills-refactor\references\clean-code-rules.md`.
Si 404, cherche la même URL avec `mattpocock` au lieu de `ciembor`.
Si les deux échouent, note-le et construis un fichier de règles synthétiques
depuis ta connaissance de Clean Code (Robert C. Martin).

Ajoute en tête du fichier récupéré :

```
# Clean code rules — source : ciembor/agent-rules-books
# Utilisé par : principles-auditor
# Licence : MIT
---
```

### 1b. Récupérer refactoring.mini.md

Fetch :
`https://raw.githubusercontent.com/ciembor/agent-rules-books/main/refactoring/refactoring.mini.md`

Enregistre dans `references\refactoring-rules.md` avec le même en-tête.

### 1c. Vérifier si d'autres règles pertinentes existent

Fetch la liste du repo :
`https://api.github.com/repos/ciembor/agent-rules-books/contents`

Si tu vois `clean-architecture` ou `working-effectively-with-legacy-code`,
fetch également leurs `.mini.md` et note-les en bas de `references\clean-code-rules.md`
avec la mention `# Supplément — Clean Architecture`.

### 1d. Explorer sober-coding pour les AI smells

Fetch : `https://raw.githubusercontent.com/voidborne-d/sober-coding/main/README.md`

Si 200 : extrais les 27 règles listées et leurs descriptions.
Si 404 : cherche `sober-coding` sur GitHub via WebSearch, tente une autre URL.
Si introuvable : construis `references\ai-smells.md` avec les 27 patterns
AI-générés connus (over-generation, structural clones, god files, swallowed errors,
magic values, dual state sources, optimistic UI drift, stale closures,
mutation directe, useEffect deps incorrects, etc.) depuis ta connaissance du domaine.

---

## Phase 2 — Construction des fichiers de références internes

### 2a. references\safe-refactor.md

Crée ce fichier avec les règles de sécurité pour le refactor-executor.
Il doit couvrir :

**Ce qui est toujours sûr (sans confirmation) :**
- Supprimer du code mort confirmé (0 importeurs, vérifié cross-fichiers)
- Supprimer les `console.log` / `dbg!` / `print!` en dehors des tests
- Supprimer les blocs commentés > 3 lignes
- Extraire une constante nommée pour un magic number dans le même fichier
- Ajouter/corriger un JSDoc ou docstring (pas de changement de signature)

**Sûr avec validation post-modification :**
- Renommer un symbole exporté + mettre à jour tous les importeurs dans le même passage
- Extraire une fonction dans le même fichier
- Remplacer `array.push(x)` par `setArray([...array, x])` dans un contexte React
- Ajouter des types TypeScript manquants (doit passer tsc)
- Déplacer une fonction vers un fichier utils + mettre à jour les imports

**Jamais sans confirmation explicite :**
- Changer une signature de fonction publique
- Modifier le type de retour d'une fonction
- Supprimer un fichier entier
- Modifier les fichiers d'authentification / sécurité
- Toucher les migrations / schéma base de données
- Modifier les fichiers de tests au-delà de la mise à jour des imports

**Jamais toucher :**
- Fichiers générés (`dist/`, `build/`, `*.generated.*`)
- Lock files
- Fichiers dans `violations.manual_verify`
- Tout fichier marqué `// DO NOT EDIT`

**Protocole par fichier :**
1. Appeler `qartez_impact` si le fichier est dans `findings.hotspots` avec score > 5
2. Appliquer le changement
3. Valider : `npx tsc --noEmit && npx biome check <fichier>`
4. Si échec → revert + log dans `changes.skipped` avec raison
5. Passer au fichier suivant

### 2b. references\principles.md

Crée ce fichier avec les 10 principes fondamentaux. Formule chaque principe
avec : la règle, les violations détectables, le fix standard.

Couvre obligatoirement :
1. Single responsibility (fichier, fonction, module)
2. Single source of truth (état, données, config)
3. Immutabilité dans les contextes réactifs (React, Vue, Zustand)
4. Contrats typés aux frontières (API, IPC, formulaires, env vars)
5. Erreurs explicites (pas de catch vide, pas de continue silencieux)
6. UI pure (render = f(state), pas d'effets de bord dans le render)
7. DRY avec seuil pragmatique (2× → watch, 3× → extract)
8. Nommage intentionnel (verbes pour fonctions, is/has pour booléens)
9. Fonctions petites et lisibles (≤40 lignes, ≤3 niveaux, early return)
10. Documentation vivante (JSDoc sur exports, AGENTS.md à jour)

Pour chaque principe, ajoute une section "Détection automatique" listant
ce que Qartez, Knip, Lizard, Biome, ou Clippy peut détecter à sa place.

---

## Phase 3 — Construction des agents

Chaque fichier agent est un fichier `.md` avec frontmatter YAML + corps d'instructions.

### 3a. agents\scout.md

```yaml
---
name: scout
description: >
  Cartographie objective du codebase via outils déterministes.
  Appelle Knip, Lizard, Biome, et Cargo Clippy pour produire findings.json.
  Utilise les MCP Qartez et @knip/mcp en option pour enrichir l'analyse.
  Ne modifie aucun fichier. Read-only strict.
tools: Read, Glob, Grep, Bash
model: sonnet
---
```

Corps du fichier — instructions complètes pour le scout :

Tu es un agent d'analyse statique. Tu cartographies le codebase de façon objective
et reproductible. Tu ne portes aucun jugement qualitatif — tu collectes des faits.

**Comportement strict :**
- Ne modifie aucun fichier. Si tu te surprends à vouloir écrire, arrête-toi.
- Toutes tes sorties vont dans `.claude/quality-team/findings.json`.
- Si un outil CLI échoue, log l'erreur dans findings.errors et continue.

**Séquence d'analyse :**

Étape 1 — Si Qartez MCP est disponible, appelle dans cet ordre :
- `qartez_map` sur le scope → vue globale du projet
- `qartez_hotspots` → fichiers risqués (score = complexité × PageRank × churn)
- `qartez_unused` → exports sans importeurs (dead code cross-fichiers)
- `qartez_clones` → duplications structurelles via AST hashing
- `qartez_deps` sur les 5 fichiers les plus importants → graphe de dépendances

Si Qartez n'est pas disponible, note `"qartez": "unavailable"` dans findings et continue.

Étape 2 — Appelle les CLI déterministes (obligatoires) :

```bash
# Dead code et dépendances inutilisées (TypeScript/JS)
npx knip --reporter json 2>/dev/null > .claude/quality-team/knip_raw.json

# Complexité cyclomatique par fonction
lizard $SCOPE --CCN 10 --length 40 --arguments 5 -o .claude/quality-team/lizard_raw.json 2>/dev/null

# Lint TypeScript/JS
npx biome check $SCOPE --reporter json 2>/dev/null > .claude/quality-team/biome_raw.json

# Lint Rust (si src-tauri/ ou *.rs existent)
if (Test-Path "src-tauri") { cargo clippy --message-format json 2>/dev/null > .claude/quality-team/clippy_raw.json }
```

Étape 3 — Si @knip/mcp est disponible, utilise-le pour affiner les résultats
Knip (il peut distinguer les exports réellement inutilisés des faux positifs
liés aux re-exports et aux barrel files).

Étape 4 — Consolide tout dans findings.json :

```json
{
  "scope": "<chemin analysé>",
  "generated_at": "<ISO timestamp>",
  "tools_used": ["knip", "lizard", "biome", "qartez"],
  "hotspots": [
    {
      "file": "src/...",
      "score": 8.2,
      "reason": "high_ccn+churn+pagerank",
      "blast_radius": 14
    }
  ],
  "dead_code": [
    {
      "file": "src/...",
      "symbol": "oldHandler",
      "type": "function|export|file",
      "tool": "knip|qartez"
    }
  ],
  "complexity": [
    {
      "file": "src/...",
      "fn": "processQueue",
      "ccn": 18,
      "nloc": 67,
      "params": 4,
      "threshold_breached": "ccn>10"
    }
  ],
  "clones": [
    {
      "files": ["src/a.ts", "src/b.ts"],
      "lines": 12,
      "tool": "lizard|qartez"
    }
  ],
  "lint": [
    {
      "file": "src/...",
      "line": 42,
      "rule": "noDirectMutation",
      "severity": "error",
      "message": "...",
      "tool": "biome|clippy"
    }
  ],
  "unused_deps": [
    {
      "package": "some-lib",
      "type": "dep|devDep"
    }
  ],
  "errors": []
}
```

Enregistre findings.json dans `.claude/quality-team/findings.json`.
Écris une ligne de résumé dans stdout : nombre de hotspots, dead code, violations lint.

### 3b. agents\principles-auditor.md

```yaml
---
name: principles-auditor
description: >
  Analyse qualitative du codebase en appliquant les règles clean code, refactoring
  et AI smells. Lit findings.json produit par scout + les fichiers réels flaggés.
  Utilise Qartez MCP en option pour comprendre le contexte avant de juger.
  Ne modifie aucun fichier. Produit violations.json avec severity labeling.
tools: Read, Grep
model: sonnet
---
```

Corps — charge les règles depuis les fichiers references/ qui sont disponibles
dans le contexte projet. Si tu trouves `references/clean-code-rules.md`,
`references/refactoring-rules.md`, `references/ai-smells.md`, et
`references/principles.md`, lis-les avant de commencer l'analyse.

**Comportement strict :**
- Ne modifie aucun fichier.
- Avant de flagguer un symbole comme "inutilisé", vérifie avec `qartez_refs`
  s'il a des usages dynamiques que Knip n'aurait pas vus.
- Avant de flagguer un fichier comme "trop long", lis-le avec `qartez_outline`
  pour comprendre sa structure réelle.
- Si tu es incertain sur une violation, mets-la dans `suggestion` pas `blocking`.

**Séquence :**

1. Lis `findings.json` complet.
2. Pour les 10 fichiers les plus critiques (hotspots + lint violations) :
   - Lis le fichier
   - Si Qartez disponible : appelle `qartez_outline` pour la structure globale
   - Applique les règles de `references/principles.md` (nos 10 principes)
   - Applique les règles de `references/clean-code-rules.md` (Fowler/Martin)
   - Applique les règles de `references/refactoring-rules.md`
   - Cherche les patterns de `references/ai-smells.md` (27 AI smells)
3. Pour les dead code identifiés par scout :
   - Vérifie avec `qartez_refs` avant de confirmer (faux positifs possibles)
4. Classe chaque finding avec une sévérité :
   - `blocking` : cause des bugs, viole un principe fondamental, unsafe
   - `important` : code difficile à maintenir, dette significative
   - `nit` : style, nommage, petit cleanup
   - `suggestion` : amélioration optionnelle, pas urgent
   - `manual_verify` : changement trop risqué sans couverture de tests

Produit `violations.json` :

```json
{
  "generated_at": "<ISO timestamp>",
  "files_analyzed": 10,
  "blocking": [
    {
      "file": "src/stores/queue.ts",
      "line": 42,
      "principle": "P2-single-source-of-truth",
      "description": "State dupliqué : Zustand mirror ne se resynchronise jamais depuis le backend Rust",
      "evidence": "queueStore.entries mis à jour optimistement sans reconciliation",
      "fix_hint": "Remplacer setEntries([...entries, newEntry]) par setEntries(result.entries) après chaque commande IPC"
    }
  ],
  "important": [],
  "nit": [],
  "suggestion": [],
  "ai_smells": [
    {
      "file": "src/...",
      "rule": "structural-clone",
      "description": "Logique de formatage copiée identiquement dans 3 composants"
    }
  ],
  "manual_verify": [
    {
      "file": "src/auth/handler.ts",
      "reason": "Pas de couverture de tests — refactoring à risque",
      "recommendation": "Ajouter des tests avant de toucher ce fichier"
    }
  ]
}
```

### 3c. agents\refactor-executor.md

```yaml
---
name: refactor-executor
description: >
  Applique les changements sûrs identifiés par principles-auditor.
  Traite uniquement les findings blocking + important avec safety gate.
  Valide avec tsc/biome/clippy après chaque fichier. Si validation échoue,
  revert et log dans skipped. Ne touche jamais manual_verify.
tools: Read, Write, Edit, MultiEdit, Bash
model: sonnet
skills:
  - safe-refactor-rules
---
```

Corps — lis `references/safe-refactor.md` au démarrage (disponible dans le
contexte projet) avant de toucher quoi que ce soit.

**Protocole strict par fichier :**

```
Pour chaque fichier dans violations.blocking + violations.important :

1. Si fichier dans violations.manual_verify → SKIP, log dans changes.skipped

2. Si fichier dans findings.hotspots avec blast_radius > 5 ET Qartez disponible :
   → appelle qartez_impact(fichier)
   → si impact.transitive_dependents > 20 → SKIP sauf si fix trivial (dead code)
   → note le blast radius dans changes.applied

3. Si le changement est un rename cross-fichiers ET Qartez disponible :
   → appelle qartez_refs(symbole) pour lister tous les usages
   → renomme dans tous les fichiers en un seul passage avec MultiEdit

4. Applique le changement (le plus petit changement possible)

5. Validation :
   → npx tsc --noEmit (si fichier TypeScript)
   → npx biome check <fichier> (si fichier JS/TS)
   → cargo clippy (si fichier Rust)
   
6. Si validation OK → log dans changes.applied avec validated: true
   Si validation KO → git diff pour voir ce qui a changé → revert → log dans changes.skipped avec raison

7. Passer au fichier suivant
```

Produit `changes.json` :

```json
{
  "generated_at": "<ISO timestamp>",
  "applied": [
    {
      "file": "src/...",
      "type": "dead-code-removal|rename|extract-constant|immutability-fix|error-handling",
      "description": "Supprimé export 'legacyHandler' (0 importeurs, confirmé qartez_refs)",
      "blast_radius_checked": true,
      "validated": true,
      "tools_passed": ["tsc", "biome"]
    }
  ],
  "skipped": [
    {
      "file": "src/...",
      "reason": "reverted:tsc-failed|manual-verify|blast-radius-too-high",
      "validation_output": "error TS2345: ...",
      "recommendation": "Ajouter des tests, puis retry avec /quality-team"
    }
  ],
  "baseline": {
    "tsc": "pass|fail",
    "biome": "pass|fail",
    "clippy": "pass|fail|skipped"
  }
}
```

### 3d. agents\doc-updater.md

```yaml
---
name: doc-updater
description: >
  Met à jour la documentation pour refléter chaque changement appliqué par
  refactor-executor. Ajoute ou corrige JSDoc sur les fonctions modifiées,
  met à jour AGENTS.md si des modules ont bougé, et produit le REFACTOR_REPORT.md
  final. Ne modifie jamais le code source.
tools: Read, Write, Edit
model: sonnet
skills:
  - code-doc-architect
---
```

Corps — lis `changes.json` et la liste des fichiers modifiés. Pour chaque
fichier dans `changes.applied` :

1. Lis le fichier modifié
2. Pour chaque fonction/export modifié, vérifie si le JSDoc est à jour :
   - Paramètres correspondent à la signature actuelle ?
   - `@returns` décrit le bon type ?
   - `@throws` mentionne les cas d'erreur si applicable ?
   - Ajoute un commentaire `// why:` si le changement n'est pas évident
3. Si un fichier a été renommé ou déplacé, mets à jour AGENTS.md
4. Si une commande dans README.md ne correspond plus → mets-la à jour
5. Génère `REFACTOR_REPORT.md` à la racine du projet analysé

Template REFACTOR_REPORT.md :

```markdown
# Rapport de refactoring — <date>

## Résumé exécutif
- Fichiers analysés : N
- Violations bloquantes trouvées : N (N corrigées, N skipped)
- Violations importantes : N (N corrigées, N skipped)
- AI smells détectés : N
- Code mort supprimé : N symboles
- Baseline post-refactor : tsc ✅ · biome ✅ · clippy ✅

## Corrections appliquées
### Bloquant
| Fichier | Changement | Type | Validé |
|---------|-----------|------|--------|
| ...     | ...       | ...  | ✅     |

### Important
...

## Éléments non traités (vérification manuelle requise)
| Fichier | Raison | Recommandation |
|---------|--------|---------------|
| ...     | ...    | ...           |

## AI smells identifiés
...

## Documentation mise à jour
- JSDoc : N fonctions mises à jour
- AGENTS.md : [oui/non]
- README.md : [oui/non]

## Prochaines étapes
1. Ajouter des tests sur les fichiers dans manual_verify
2. Relancer /quality-team pour un second passage
3. Configurer les MCP manquants (voir MCP_CHECKLIST.md)
```

---

## Phase 4 — Construction du skill orchestrateur

### quality-team\SKILL.md

```yaml
---
name: quality-team
description: >
  Lance une chaîne de 4 sous-agents pour analyser, nettoyer, et documenter
  un codebase. Spawn séquentiellement : scout (analyse statique) →
  principles-auditor (règles clean code + AI smells) → refactor-executor
  (corrections sûres avec validation) → doc-updater (JSDoc + rapport final).
  Use when: "nettoie mon code", "audit qualité", "refactor", "code mort",
  "trop de bugs pour pas de raison". Do not use: ajout de features, corrections
  de bugs spécifiques, génération de tests from scratch.
when_to_use: >
  Déclencher sur : "quality-team", "nettoie le code", "audit", "refactor",
  "code mort", "mauvaise qualité", "trop de problèmes structurels".
  Ne pas déclencher sur : ajout de feature, fix de bug précis, écriture de tests.
argument-hint: "[scope] [mode]"
arguments:
  - scope     # chemin relatif à analyser (défaut: src/)
  - mode      # "audit-only" | "refactor" | "all" (défaut: all)
allowed-tools:
  - Read
  - Bash(New-Item *)
  - Bash(mkdir *)
  - Bash(npx tsc --noEmit *)
  - Bash(npx biome check *)
  - Bash(cargo clippy *)
user-invocable: true
---
```

Corps du SKILL.md orchestrateur :

## Comportement

Lis le scope et le mode depuis `$ARGUMENTS`. Défaut : `scope=src/`, `mode=all`.

Prépare le répertoire de travail :

```bash
New-Item -ItemType Directory -Force -Path ".claude/quality-team"
```

**Phase 0 — Baseline**

Avant de spawner quoi que ce soit, enregistre les métriques de base :

```bash
npx tsc --noEmit 2>&1 | Tee-Object -FilePath .claude/quality-team/baseline_tsc.txt
npx biome check $scope --reporter json 2>/dev/null | ConvertTo-Json | Out-File .claude/quality-team/baseline_biome.json
```

**Phase 1 — Scout**

Spawne le sous-agent `scout` avec ce prompt :

```
Analyse le scope : <scope>.
Lis tes instructions dans agents/scout.md.
Produit : .claude/quality-team/findings.json
Mode : lecture seule, aucune modification de fichier.
```

Attends la fin. Vérifie que `.claude/quality-team/findings.json` existe.
Si absent → arrête avec message d'erreur clair.

**Phase 2 — Principles auditor**

Spawne le sous-agent `principles-auditor` avec ce prompt :

```
Lis .claude/quality-team/findings.json (produit par scout).
Lis les fichiers dans references/ disponibles dans ce projet.
Analyse les fichiers flaggés et applique les règles clean code.
Produit : .claude/quality-team/violations.json
Mode : lecture seule, aucune modification de fichier.
```

Attends la fin. Vérifie que `.claude/quality-team/violations.json` existe.
Affiche un résumé : N blocking, N important, N ai_smells.

**Phase 3 — Refactor executor** (seulement si mode ≠ "audit-only")

Spawne `refactor-executor` avec ce prompt :

```
Lis .claude/quality-team/violations.json (produit par principles-auditor).
Lis .claude/quality-team/findings.json pour les données de hotspots.
Lis references/safe-refactor.md avant toute modification.
Applique uniquement les findings blocking + important qui passent le safety gate.
Ne touche JAMAIS les fichiers dans violations.manual_verify.
Valide avec tsc/biome/clippy après chaque fichier modifié.
Si validation échoue : revert et log dans changes.skipped.
Produit : .claude/quality-team/changes.json
```

Attends la fin. Vérifie `.claude/quality-team/changes.json`.

**Phase 4 — Doc updater**

Spawne `doc-updater` avec ce prompt :

```
Lis .claude/quality-team/changes.json (produit par refactor-executor).
Met à jour le JSDoc des fonctions modifiées.
Si des modules ont été déplacés, mets à jour AGENTS.md.
Génère REFACTOR_REPORT.md à la racine du projet analysé.
Ne modifie jamais le code source.
```

Attends la fin.

**Phase 5 — Validation finale**

```bash
npx tsc --noEmit
npx biome check $scope --reporter json
```

Compare avec la baseline. Affiche : tests ✅/❌, lint ✅/❌.

**Synthèse finale**

Affiche :
- Chemin du rapport : `REFACTOR_REPORT.md`
- Résumé des changements appliqués vs skipped
- Fichiers à traiter manuellement (violations.manual_verify)
- Rappel : voir `MCP_CHECKLIST.md` pour améliorer les analyses futures

---

## Phase 5 — Schémas JSON

### quality-team\schemas\findings.schema.json

Crée un JSON Schema 2020-12 validant la structure de findings.json.
Champs requis : scope, generated_at, tools_used (array), hotspots (array),
dead_code (array), complexity (array), clones (array), lint (array),
unused_deps (array), errors (array).

### quality-team\schemas\violations.schema.json

JSON Schema pour violations.json.
Champs requis : generated_at, files_analyzed, blocking (array), important (array),
nit (array), suggestion (array), ai_smells (array), manual_verify (array).

### quality-team\schemas\changes.schema.json

JSON Schema pour changes.json.
Champs requis : generated_at, applied (array), skipped (array),
baseline (object avec tsc, biome, clippy).

---

## Phase 6 — Playbooks stack-spécifiques

### playbooks\react-ts.md

Crée un playbook ciblant React + TypeScript + Tauri. Il doit couvrir :

**State management :**
- Détection de `array.push()` / `obj.field =` sur du state React → violation P3
- État dupliqué entre store Zustand et backend Tauri → violation P2
- `useEffect` sans deps array approprié → violation (règle react-hooks)
- `useMemo` / `useCallback` manquants sur des props objet/fonction instables

**Tauri IPC :**
- `file.path || file.name` pour obtenir un chemin → toujours faux en WebView2
- Réponse IPC non validée par Zod → violation P4
- Pas de resync depuis backend après mutation IPC → violation P2

**TypeScript :**
- `any` sans commentaire justificatif → violation P4
- `as SomeType` sur une réponse externe → violation P4
- `tsconfig.json` sans `strict: true` → recommandation

**Hooks React :**
- Hook qui fait plus d'une chose → violation P1
- `useEffect` avec side effect sans cleanup → violation
- State non localisé au plus bas niveau utile → violation P2

**Outils de détection :**
- biome: noDirectMutation, useExhaustiveDependencies
- tsc strict pour les types
- qartez_hotspots pour les fichiers à fort couplage

### playbooks\rust.md

Crée un playbook ciblant Rust + Tauri backend. Couvre :

**Gestion d'erreurs :**
- `unwrap()` / `expect()` sur des chemins utilisateur → violation P5
- `let _ = result_que_peut_echouer` → violation P5
- `catch { continue }` implicite en Rust → violation P5
- Toute commande `#[tauri::command]` sans retour `Result<T, E>` → violation

**Nommage et structure :**
- Convention snake_case
- `pub use` dans `mod.rs` pour une API propre
- Pas de logique dans `mod.rs`

**Clippy obligatoire :**
- `clippy::unwrap_used`, `clippy::expect_used`, `clippy::panic`
- `cargo clippy -- -D warnings` en CI

**Paths et fichiers :**
- `PathBuf::from("nom.pdf").exists()` sans chemin absolu → toujours faux
- Chemins construits depuis une string sans validation → violation P4

---

## Phase 7 — Templates

### templates\audit-report.md

Crée le template complet de REFACTOR_REPORT.md avec tous les placeholders
clairement nommés. Inclure les sections :
résumé exécutif, métriques baseline vs post-refactor, corrections appliquées
(par sévérité), éléments skipped avec recommandations, AI smells détectés,
documentation mise à jour, prochaines étapes.

---

## Phase 8 — MCP_CHECKLIST.md

Crée ce fichier à la racine de `C:\Projets\skills-refactor`.
Il est destiné à l'utilisateur qui configurera les MCP manuellement.

```markdown
# Checklist MCP — quality-refactor-team

Ces serveurs MCP améliorent l'analyse mais ne sont pas obligatoires.
La chaîne d'agents fonctionne sans eux (via les CLI déterministes).

## MCP servers à ajouter dans ~/.claude/settings.json

### 1. Qartez MCP (intelligence structurelle)
- **Pourquoi** : hotspots, blast radius, dead code cross-fichiers, clones AST,
  call graph. Réduit les tokens de ~94%. Utilisé par scout, principles-auditor,
  refactor-executor.
- **Install** :
  ```bash
  # macOS / Linux
  curl -sSfL https://qartez.dev/install | sh
  # Windows PowerShell
  powershell -ExecutionPolicy Bypass -c "iwr https://raw.githubusercontent.com/kuberstar/qartez-mcp/main/install.ps1 -useb | iex"
  ```
- **Config settings.json** :
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
- **Outils utilisés par les agents** :
  - scout : qartez_map, qartez_hotspots, qartez_unused, qartez_clones, qartez_deps
  - principles-auditor : qartez_read, qartez_outline, qartez_refs, qartez_calls
  - refactor-executor : qartez_impact (OBLIGATOIRE avant tout fichier hotspot > 5)

### 2. @knip/mcp (dead code TypeScript)
- **Pourquoi** : détecte les exports/fichiers/deps inutilisés en cross-fichiers,
  comprend les barrel files et les framework entry points. Complète Qartez.
- **Install** : `npm install -D knip`
- **Config settings.json** :
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
- **Agents qui l'utilisent** : scout (enrichit knip_raw.json), principles-auditor

### 3. SonarQube MCP (optionnel, qualité enterprise)
- **Pourquoi** : code smells, duplication, sécurité, quality gates. Gratuit en
  Community Build self-hosted.
- **Install** : Docker self-hosted + MCP server officiel SonarSource
- **Agents qui l'utilisent** : scout (comme source additionnelle)

## CLI tools à installer globalement

Ces outils sont appelés directement par les agents via Bash — pas via MCP.
Ils doivent être dans le PATH.

| Outil | Install | Utilisé par |
|-------|---------|------------|
| Biome | `npm install -g @biomejs/biome` | scout, refactor-executor |
| Knip | `npm install -D knip` (par projet) | scout |
| Lizard | `pip install lizard` | scout |
| TypeScript | `npm install -D typescript` (par projet) | refactor-executor |

## Vérification rapide

```bash
# Vérifie que les CLI sont disponibles
npx biome --version
npx knip --version
lizard --version
npx tsc --version
cargo --version
qartez --version  # si Qartez installé
```
```

---

## Phase 9 — INSTALL.md

Crée ce fichier à la racine de `C:\Projets\skills-refactor`.

Instructions :
1. Prérequis (Node.js 20+, Python 3.8+, Rust si projets Tauri)
2. Copier le dossier `quality-team/` dans `~/.claude/skills/`
3. Copier le dossier `agents/` dans `~/.claude/agents/`
4. Copier `references/` et `playbooks/` dans le projet à analyser sous `.claude/`
5. Ajouter les MCP depuis `MCP_CHECKLIST.md`
6. Utilisation : `/quality-team src/` ou `/quality-team . audit-only`

---

## Phase 10 — quality-team\README.md

Crée un README clair couvrant :
- Ce que fait la chaîne (une phrase par agent)
- Structure des fichiers produits
- Commandes d'utilisation
- Modes disponibles (audit-only, refactor, all)
- Limitations connues (fichiers sans tests → manual_verify)
- Lien vers MCP_CHECKLIST.md pour améliorer l'analyse

---

## Phase 11 — Validation finale

Une fois tous les fichiers créés :

1. Liste tous les fichiers créés avec leur taille :
   ```bash
   Get-ChildItem -Recurse "C:\Projets\skills-refactor" | Select-Object FullName, Length
   ```

2. Vérifie que chaque fichier agent a un frontmatter YAML valide :
   - name, description, tools présents
   - tools est une liste de valeurs valides

3. Vérifie que quality-team\SKILL.md a :
   - name, description, when_to_use, argument-hint
   - allowed-tools
   - user-invocable: true

4. Vérifie que les 3 schémas JSON sont valides JSON :
   ```bash
   Get-Content "C:\Projets\skills-refactor\quality-team\schemas\findings.schema.json" | ConvertFrom-Json
   ```

5. Affiche un récapitulatif final :
   ```
   ✅ quality-team/SKILL.md
   ✅ agents/scout.md
   ✅ agents/principles-auditor.md
   ✅ agents/refactor-executor.md
   ✅ agents/doc-updater.md
   ✅ references/clean-code-rules.md  (source: ciembor/agent-rules-books OU synthétique)
   ✅ references/refactoring-rules.md (source: ciembor/agent-rules-books OU synthétique)
   ✅ references/ai-smells.md         (source: sober-coding OU synthétique)
   ✅ references/safe-refactor.md
   ✅ references/principles.md
   ✅ playbooks/react-ts.md
   ✅ playbooks/rust.md
   ✅ templates/audit-report.md
   ✅ quality-team/schemas/findings.schema.json
   ✅ quality-team/schemas/violations.schema.json
   ✅ quality-team/schemas/changes.schema.json
   ✅ MCP_CHECKLIST.md
   ✅ INSTALL.md
   ✅ quality-team/README.md
   
   Prochaine étape : voir MCP_CHECKLIST.md pour configurer les serveurs MCP.
   Usage : copier quality-team/ dans ~/.claude/skills/ et agents/ dans ~/.claude/agents/
   ```

---

## Notes importantes

- Si une URL GitHub retourne 404, construis le fichier depuis ta connaissance
  du domaine et note `# Source: synthétique — URL inaccessible` en tête.
- Ne demande pas de confirmation entre les phases. Avance de façon autonome.
- Si un fichier existe déjà dans `C:\Projets\skills-refactor`, écrase-le
  (on construit from scratch).
- Les schemas JSON doivent être complets et valides, pas des squelettes vides.
- Le SKILL.md orchestrateur doit être exécutable tel quel — pas un template
  avec des placeholders à remplir.
- Le contenu de `references/clean-code-rules.md` récupéré depuis ciembor
  doit être conservé tel quel sans résumé — c'est la densité des règles
  qui donne la valeur au principles-auditor.

# Mission : quality-team généraliste

Construire et maintenir un skill `quality-team` utilisable sur n'importe quel
projet de code. Le coeur du skill ne doit pas supposer React, Tauri, TypeScript,
Rust, npm, Biome, Clippy ou tout autre outil spécifique.

## Architecture cible

Chaîne d'agents :

```text
scout -> principles-auditor -> refactor-executor -> doc-updater
```

Le skill orchestrateur reste un routeur :
- parse `scope` et `mode`
- détecte le profil projet
- charge les références universelles
- charge les playbooks optionnels seulement si applicables
- spawne les agents
- présente un plan avant toute modification
- synthétise le résultat final

## Principe général

Le pipeline doit fonctionner même si aucun outil spécialisé n'est disponible.
Qartez, Lizard, linters, typecheckers et test runners enrichissent l'analyse,
mais ne sont pas requis globalement.

Si aucune validation n'est détectée :
- l'audit continue
- le plan indique `validation: skipped`
- le refactor automatique reste plus conservateur

## Fichiers principaux

```text
quality-team/
  SKILL.md
  README.md
  references/
    principles.md
    safe-refactor.md
    ai-smells.md
    clean-code-rules.md
    refactoring-rules.md
  playbooks/
    react-ts.md
    rust.md
  templates/
    audit-report.md
    refactor-plan.md
  schemas/
    findings.schema.json
    violations.schema.json
    changes.schema.json

agents/
  scout.md
  principles-auditor.md
  refactor-executor.md
  doc-updater.md
```

## Références universelles

`references/principles.md` contient les règles applicables à tout codebase :
responsabilité, source de vérité, contrats, invariants, erreurs, effets de bord,
duplication, nommage, complexité et documentation vivante.

`references/safe-refactor.md` définit uniquement les refactors sûrs et
universels : suppression de code mort confirmé, logs de debug, blocs commentés,
constantes locales, renommages confirmés, petites extractions locales et
documentation.

`references/ai-smells.md` décrit les patterns de code généré par IA sans dépendre
d'un framework.

## Playbooks optionnels

Les playbooks sont des extensions stack-spécifiques. Ils ne sont chargés que si
le projet détecté correspond :
- `react-ts.md` : React / TypeScript / Tauri
- `rust.md` : Rust

Un playbook non applicable ne doit jamais produire de violation.

## Sorties JSON

`findings.json` contient :
- scope
- profil projet détecté
- outils utilisés
- commandes de validation candidates
- hotspots, dead code, complexité, clones, lint, dépendances inutilisées, erreurs

`violations.json` contient :
- blocking, important, nit, suggestion
- ai_smells
- manual_verify

`changes.json` contient :
- applied
- skipped
- validation_results sous forme générique :

```json
[
  {
    "name": "test",
    "command": "npm test",
    "status": "pass",
    "output_path": ".claude/quality-team/test.out"
  }
]
```

## Validation

La validation est auto-détectée :
- scripts déclarés par le projet : `test`, `lint`, `typecheck`, `check`, `build`
- commandes natives si manifest/config présent
- sinon `skipped`

Le refactor-executor utilise uniquement les commandes détectées dans
`.claude/quality-team/validation_commands.json`.

## Consultation utilisateur

Avant tout refactor, le skill génère `.claude/quality-team/refactor_plan.md`
depuis `templates/refactor-plan.md`, l'affiche à l'utilisateur et attend une
validation explicite.

Sans validation explicite, aucun code n'est modifié et le run continue en rapport
uniquement.

## Documentation

`README.md`, `INSTALL.md` et `MCP_CHECKLIST.md` doivent présenter le skill comme
généraliste. Les outils stack-spécifiques doivent toujours être décrits comme
optionnels selon le projet cible.

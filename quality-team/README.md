# quality-team — Audit et refactoring prudent pour tout codebase

`/quality-team` lance une chaîne de 4 sous-agents généralistes. Le skill détecte
le projet au runtime, charge les règles universelles, puis ajoute seulement les
playbooks qui correspondent au stack détecté.

## Chaîne d'agents

| Agent | Rôle | Output |
|-------|------|--------|
| scout | Détecte manifests, langages, outils, hotspots, complexité, dead code, clones et lint best-effort. | `findings.json` |
| principles-auditor | Applique les principes universels, AI smells et playbooks applicables. | `violations.json` |
| refactor-executor | Applique uniquement les changements validés par l'utilisateur et autorisés par les règles de sécurité. | `changes.json` |
| doc-updater | Produit le rapport final et met à jour uniquement la documentation projet. | `REFACTOR_REPORT.md` |

## Fichiers produits

```text
.claude/quality-team/
  project_profile.json       # manifests, langages, playbooks applicables
  validation_commands.json   # commandes détectées pour ce projet
  baseline_validation.json   # état avant refactor
  findings.json              # analyse statique consolidée
  violations.json            # violations classifiées
  refactor_plan.md           # plan présenté avant modification
  changes.json               # changements appliqués/skippés (absent si audit-only)

REFACTOR_REPORT.md           # rapport lisible à la racine du projet analysé
```

## Utilisation

```text
/quality-team .
/quality-team src audit-only
/quality-team packages/api refactor
```

En modes `refactor` et `all`, le skill présente toujours un plan avant toute
modification. Le refactoring ne démarre qu'après validation explicite.

## Généraliste par défaut

Le coeur ne dépend d'aucun stack. Il peut analyser un projet JavaScript,
TypeScript, Rust, Python, Go, Java, PHP, Ruby, .NET ou mixte tant que des fichiers
source existent. Si aucune commande de validation n'est détectée, le pipeline
continue et marque la validation comme `skipped`.

## Playbooks optionnels

Les playbooks ajoutent des règles spécialisées seulement quand le projet le justifie :

- `playbooks/react-ts.md` : React / TypeScript / Tauri détecté
- `playbooks/rust.md` : Rust détecté

Un playbook non applicable ne doit jamais produire de violation.

## Limitations

- Les fichiers sans contexte suffisant ou trop risqués sont placés en `manual_verify`.
- Les corrections automatiques restent volontairement petites.
- Les validations dépendent des scripts et outils présents dans le projet cible.
- Qartez améliore fortement la précision, mais le skill fonctionne sans MCP.

## Architecture

```text
quality-team/
  SKILL.md
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

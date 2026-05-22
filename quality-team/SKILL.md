---
name: quality-team
description: >
  Lance une chaîne de sous-agents généralistes pour auditer, planifier,
  refactorer prudemment et documenter n'importe quel codebase. Le skill détecte
  le stack au runtime, charge seulement les playbooks pertinents, puis spawn :
  scout → principles-auditor → refactor-executor → doc-updater.
  Use when: "audit qualité", "nettoie le code", "refactor", "code mort",
  "trop de problèmes structurels". Do not use: ajout de feature, correction
  d'un bug précis, génération de tests from scratch.
when_to_use: >
  Déclencher sur : "quality-team", "audit", "refactor", "code mort",
  "mauvaise qualité", "nettoie le code", "problèmes structurels".
  Ne pas déclencher sur : ajout de feature, fix de bug précis, écriture de tests.
argument-hint: "[scope] [mode]"
arguments:
  - scope     # chemin relatif à analyser (défaut: .)
  - mode      # "audit-only" | "refactor" | "all" (défaut: all)
allowed-tools:
  - Read
  - Bash
user-invocable: true
---

## Rôle du skill

Tu es l'orchestrateur. Reste léger : détecte le contexte, charge les références,
spawne les agents, vérifie les fichiers de sortie et présente les décisions à
l'utilisateur. Ne porte pas la logique d'audit détaillée dans ce fichier.

Lis le scope et le mode depuis `$ARGUMENTS`.

- Si `$ARGUMENTS` est vide : `scope=.`, `mode=all`
- Si un seul argument : c'est le scope, `mode=all`
- Si deux arguments : premier=scope, deuxième=mode
- Modes valides : `audit-only`, `refactor`, `all`

Remplace toujours `<scope>` et `<mode>` par leurs valeurs littérales dans les
prompts envoyés aux sous-agents.

## Phase 0 — Préparation et détection

Crée `.claude/quality-team` si absent.

Détecte le projet sans supposer de stack :
- manifests : `package.json`, `Cargo.toml`, `pyproject.toml`, `setup.py`,
  `requirements.txt`, `go.mod`, `pom.xml`, `build.gradle`, `Makefile`,
  `composer.json`, `Gemfile`, fichiers `.sln` / `.csproj`
- langages dominants depuis les extensions présentes dans `<scope>`
- commandes de validation disponibles : scripts `test`, `lint`, `typecheck`,
  `check`, `build`, ou commandes natives dont le manifest/config existe

Enregistre :
- `.claude/quality-team/project_profile.json`
- `.claude/quality-team/validation_commands.json`
- `.claude/quality-team/baseline_validation.json`

Si aucune validation n'est détectée, note une validation `skipped` et continue.

## Phase 0b — Références

Lis et garde disponibles pour les prompts :
- `references/principles.md`
- `references/safe-refactor.md`
- `references/ai-smells.md`
- `templates/refactor-plan.md`
- `references/clean-code-rules.md` si le contexte le permet
- `references/refactoring-rules.md` si le contexte le permet

Charge les playbooks uniquement si le profil projet les justifie :
- `playbooks/react-ts.md` seulement pour React / TypeScript / Tauri détecté
- `playbooks/rust.md` seulement pour Rust détecté

Un playbook optionnel ne doit jamais produire de violation sur un projet qui ne
correspond pas à son stack.

## Phase 1 — Scout

Spawne `scout` :

```text
Analyse le scope : <scope>.
Mode quality-team : <mode>.
Lis .claude/quality-team/project_profile.json et validation_commands.json.
Produit : .claude/quality-team/findings.json
Contrainte : lecture seule, aucune modification de fichier.
```

Attends la fin. Si `findings.json` est absent, arrête avec un message clair.

## Phase 2 — Principles auditor

Spawne `principles-auditor` avec `findings.json`, les références universelles et
les playbooks optionnels chargés en Phase 0b :

```text
Analyse le scope : <scope>.
Mode quality-team : <mode>.
Lis .claude/quality-team/findings.json.
Produit : .claude/quality-team/violations.json
Contrainte : lecture seule, aucune modification de fichier.
Applique d'abord les principes universels, puis seulement les playbooks applicables.
```

Attends la fin. Si `violations.json` est absent, arrête avec un message clair.

## Phase 2b — Plan et validation utilisateur

Construis `.claude/quality-team/refactor_plan.md` depuis
`templates/refactor-plan.md`, `findings.json`, `violations.json` et
`validation_commands.json`.

Présente le plan avant toute modification. En modes `refactor` et `all`, ne lance
jamais `refactor-executor` sans validation explicite de l'utilisateur (`oui`,
`valide`, `continue`, `lance`). Toute réponse absente, ambiguë ou négative passe
le run en rapport uniquement.

En mode `audit-only`, affiche le plan comme recommandations et saute la Phase 3.

## Phase 3 — Refactor executor

Seulement si `mode != audit-only` et si le plan a été validé, spawne
`refactor-executor` :

```text
Analyse le scope : <scope>.
Mode quality-team : <mode>.
Plan validé par l'utilisateur : oui.
Lis .claude/quality-team/refactor_plan.md.
Lis .claude/quality-team/violations.json.
Lis .claude/quality-team/findings.json.
Lis .claude/quality-team/validation_commands.json.
Produit : .claude/quality-team/changes.json
Applique uniquement les changements prévus dans refactor_plan.md et autorisés par safe-refactor.md.
Valide avec les commandes détectées, si elles existent.
Si validation échoue : revert le fichier modifié et log dans changes.skipped.
```

## Phase 4 — Doc updater

Spawne `doc-updater` :

```text
Analyse le scope : <scope>.
Mode quality-team : <mode>.
Lis findings.json, violations.json, refactor_plan.md, validation_commands.json.
Lis changes.json seulement s'il existe.
Génère REFACTOR_REPORT.md à la racine du projet analysé.
Ne modifie jamais le code source ; seules les docs peuvent changer.
```

## Phase 5 — Synthèse finale

Relance les commandes de validation détectées si un refactor a été appliqué.
Compare `baseline_validation.json` et les résultats post-refactor.

Affiche :
- scope et mode
- validation avant → après pour chaque commande détectée
- chemin du rapport
- changements appliqués / skippés
- fichiers en vérification manuelle
- playbooks chargés

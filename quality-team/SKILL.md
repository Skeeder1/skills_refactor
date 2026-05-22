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

## Comportement

Lis le scope et le mode depuis `$ARGUMENTS`.

- Si `$ARGUMENTS` est vide → `scope=src/`, `mode=all`
- Si un seul argument → c'est le scope, `mode=all`
- Si deux arguments → premier=scope, deuxième=mode
- Valeurs valides pour mode : `audit-only`, `refactor`, `all`

---

## Phase 0 — Préparation

verifier le répertoire de travail, le creer si il n'existe pas, et s'assurer que les outils nécessaires sont installés.:
Prevenir l'utilisateur si des prerequis sont manquants.
```bash
New-Item -ItemType Directory -Force -Path ".claude/quality-team"
```

Enregistre les métriques de base **avant tout changement** :

```bash
# Baseline TypeScript
npx tsc --noEmit 2>&1 | Out-File ".claude/quality-team/baseline_tsc.txt" -Encoding utf8

# Baseline Biome
npx biome check $scope --reporter json 2>$null | Out-File ".claude/quality-team/baseline_biome.json" -Encoding utf8

# Baseline Rust (si applicable)
if (Test-Path "src-tauri") {
  cargo clippy --message-format json 2>$null | Out-File ".claude/quality-team/baseline_clippy.json" -Encoding utf8
}
```

Note l'état baseline (pass/fail) — il sera comparé à la fin.

---

## Phase 0b — Chargement des références depuis le skill

Le skill possède ses propres fichiers de règles dans `references/` et `playbooks/`.
Lis-les maintenant avec l'outil `Read` (chemins relatifs depuis ce fichier SKILL.md) :

**Obligatoires :**
- `references/principles.md` → règles P1-P10
- `references/safe-refactor.md` → règles de sécurité pour l'exécuteur
- `references/ai-smells.md` → 27 patterns AI-générés

**Optionnels selon le projet :**
- `references/clean-code-rules.md` → si tu as de la place dans le contexte
- `references/refactoring-rules.md` → si tu as de la place dans le contexte
- `playbooks/react-ts.md` → si des fichiers `.tsx` sont dans le scope
- `playbooks/rust.md` → si `src-tauri/` existe dans le projet

Ces contenus seront passés **inline** dans les prompts de spawn des agents qui en ont besoin.
Les agents n'ont pas accès aux fichiers du skill — tout ce qu'ils savent vient de leur prompt.

---

## Phase 1 — Scout

Spawne le sous-agent `scout` avec ce prompt :

```
Analyse le scope : <scope>.
Produit : .claude/quality-team/findings.json
Mode : lecture seule, aucune modification de fichier.
Écris une ligne de résumé dans stdout quand tu as terminé.
```

> Note : l'agent `scout` a ses instructions complètes chargées depuis ~/.claude/agents/scout.md.
> Le prompt de spawn n'a besoin que du scope et du chemin de sortie.

**Attends la fin.**

Vérifie que `.claude/quality-team/findings.json` existe :
- Si présent → affiche : `✅ Scout terminé — findings.json généré`
- Si absent → affiche : `❌ Erreur : findings.json absent. Le scout a échoué.` et **arrête**.

---

## Phase 2 — Principles auditor

Spawne le sous-agent `principles-auditor` avec ce prompt (inclure le contenu des références lues en Phase 0b) :

```
Analyse le scope : <scope>.
Lis .claude/quality-team/findings.json (produit par scout).
Produit : .claude/quality-team/violations.json
Mode : lecture seule, aucune modification de fichier.

=== PRINCIPES FONDAMENTAUX (P1-P10) ===
<inclure ici le contenu complet de references/principles.md lu en Phase 0b>

=== AI SMELLS (27 PATTERNS) ===
<inclure ici le contenu complet de references/ai-smells.md lu en Phase 0b>

[Si le contexte le permet, inclure également :]
=== CLEAN CODE RULES ===
<contenu de references/clean-code-rules.md>

=== REFACTORING RULES ===
<contenu de references/refactoring-rules.md>

[Si .tsx détecté dans le scope :]
=== PLAYBOOK REACT + TYPESCRIPT ===
<contenu de playbooks/react-ts.md>

[Si src-tauri/ existe :]
=== PLAYBOOK RUST + TAURI ===
<contenu de playbooks/rust.md>
```

**Attends la fin.**

Vérifie que `.claude/quality-team/violations.json` existe.
Lis violations.json et affiche le résumé :
```
✅ Audit terminé :
   - Blocking : N
   - Important : N
   - Nits : N
   - AI smells : N
   - Manual verify : N
```

Si `mode = "audit-only"` → **sauter les phases 3 et 4, aller directement à la Phase 5.**

---

## Phase 3 — Refactor executor (seulement si mode ≠ "audit-only")

Spawne le sous-agent `refactor-executor` avec ce prompt (inclure safe-refactor lu en Phase 0b) :

```
Analyse le scope : <scope>.
Lis .claude/quality-team/violations.json (produit par principles-auditor).
Lis .claude/quality-team/findings.json pour les données de hotspots et blast radius.
Produit : .claude/quality-team/changes.json

Applique uniquement les findings blocking + important qui passent le safety gate.
Ne touche JAMAIS les fichiers dans violations.manual_verify.
Valide avec tsc/biome/clippy après chaque fichier modifié.
Si validation échoue : revert et log dans changes.skipped avec raison détaillée.

=== RÈGLES DE SÉCURITÉ POUR LE REFACTORING ===
<inclure ici le contenu complet de references/safe-refactor.md lu en Phase 0b>
```

**Attends la fin.**

Vérifie que `.claude/quality-team/changes.json` existe.
Lis changes.json et affiche :
```
✅ Refactor terminé :
   - Changements appliqués : N
   - Fichiers skippés : N
```

---

## Phase 4 — Doc updater

Spawne le sous-agent `doc-updater` avec ce prompt :

```
Analyse le scope : <scope>.
Lis .claude/quality-team/changes.json (produit par refactor-executor).
Si mode était "audit-only", changes.json n'existe pas — génère le rapport depuis violations.json uniquement.
Met à jour le JSDoc des fonctions modifiées.
Si des modules ont été déplacés ou renommés, mets à jour AGENTS.md.
Si des commandes dans README.md ne correspondent plus, mets-les à jour.
Génère REFACTOR_REPORT.md à la racine du projet analysé.
Ne modifie jamais le code source (*.ts, *.tsx, *.rs, *.js, *.jsx).
```

**Attends la fin.**

---

## Phase 5 — Validation finale

Re-run les vérificateurs :

```bash
# Post-refactor TypeScript
npx tsc --noEmit 2>&1 | Out-File ".claude/quality-team/post_tsc.txt" -Encoding utf8

# Post-refactor Biome
npx biome check $scope --reporter json 2>$null | Out-File ".claude/quality-team/post_biome.json" -Encoding utf8
```

Compare avec la baseline et affiche :

```
═══════════════════════════════════════
  QUALITY-TEAM — RAPPORT FINAL
═══════════════════════════════════════
  Scope analysé : <scope>
  Mode : <mode>

  TypeScript  : <baseline> → <post>  [✅ amélioré / ❌ dégradé / ➡ stable]
  Biome       : <baseline> → <post>  [✅ / ❌ / ➡]
  Clippy      : <baseline> → <post>  [✅ / ❌ / ➡ / ⏭ non applicable]

  Rapport complet : REFACTOR_REPORT.md

  Changements appliqués : N  |  Skippés : N
  Fichiers à traiter manuellement : N
    → <liste des fichiers manual_verify>

  Pour améliorer l'analyse future :
    → Voir MCP_CHECKLIST.md (Qartez + @knip/mcp)
═══════════════════════════════════════
```

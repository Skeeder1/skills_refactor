# Rapport de refactoring — <date>

> Template REFACTOR_REPORT.md — utilisé par l'agent doc-updater
> Remplacer tous les placeholders `<...>` par les valeurs réelles du run.

---

## Résumé exécutif

| Métrique | Valeur |
|----------|--------|
| Scope analysé | `<scope>` |
| Mode | `<audit-only \| refactor \| all>` |
| Fichiers analysés | `<N>` |
| **Violations bloquantes** | `<N>` trouvées · `<N>` corrigées · `<N>` skipped |
| **Violations importantes** | `<N>` trouvées · `<N>` corrigées · `<N>` skipped |
| Nits identifiés | `<N>` |
| AI smells détectés | `<N>` |
| Code mort supprimé | `<N>` symboles |
| Baseline post-refactor | tsc `<✅\|❌>` · biome `<✅\|❌>` · clippy `<✅\|❌\|⏭>` |

---

## Métriques baseline vs post-refactor

| Outil | Avant refactoring | Après refactoring | Delta |
|-------|-------------------|-------------------|-------|
| tsc errors | `<N>` | `<N>` | `<+N/-N/stable>` |
| biome violations | `<N>` | `<N>` | `<+N/-N/stable>` |
| clippy warnings | `<N>` | `<N>` | `<+N/-N/skipped>` |
| Hotspots (score > 5) | `<N>` | `<N>` | `<+N/-N/stable>` |

---

## Corrections appliquées

### Bloquant

| Fichier | Changement | Type | Validé |
|---------|-----------|------|--------|
| `<src/...>` | `<description précise>` | `<dead-code-removal\|rename\|immutability-fix\|...>` | `<✅\|❌>` |

_Si aucune correction bloquante : "Aucune violation bloquante trouvée" ou "Toutes les violations bloquantes étaient en manual_verify"._

### Important

| Fichier | Changement | Type | Validé |
|---------|-----------|------|--------|
| `<src/...>` | `<description précise>` | `<type>` | `<✅\|❌>` |

_Si aucune correction importante : "Aucune correction importante appliquée"._

---

## Éléments non traités (vérification manuelle requise)

Ces fichiers n'ont pas été modifiés automatiquement. L'action recommandée est indiquée.

| Fichier | Raison du skip | Recommandation |
|---------|---------------|---------------|
| `<src/...>` | `<manual-verify\|blast-radius-too-high\|reverted:tsc-failed\|...>` | `<action à faire>` |

_Si aucun skip : "Tous les fichiers éligibles ont été traités avec succès"._

---

## AI smells identifiés

Ces patterns AI-générés ont été détectés mais ne font pas tous l'objet d'une correction automatique.
Les corrections appliquées sont marquées ✅.

| Fichier | Pattern | Description | Sévérité | Corrigé |
|---------|---------|-------------|----------|---------|
| `<src/...>` | `<structural-clone\|swallowed-errors\|...>` | `<description>` | `<blocking\|important\|nit>` | `<✅\|non>` |

---

## Documentation mise à jour

- **JSDoc** : `<N>` fonctions mises à jour dans `<liste des fichiers>`
- **AGENTS.md** : `<oui — raison>` / `<non — aucun module déplacé>`
- **README.md** : `<oui — commandes mises à jour>` / `<non — aucune référence obsolète>`

---

## Prochaines étapes

1. **Ajouter des tests** sur les fichiers dans `manual_verify` (voir tableau ci-dessus) —
   sans couverture de tests, ces fichiers ne peuvent pas être refactorisés automatiquement.
   Une fois les tests ajoutés, relancer `/quality-team <scope>` pour traiter ces fichiers.

2. **Relancer `/quality-team`** pour un second passage — les corrections de ce run ouvrent
   souvent de nouvelles opportunités de cleanup qui n'étaient pas visibles avant.

3. **Configurer les MCP manquants** pour améliorer la précision des analyses futures.
   Voir `MCP_CHECKLIST.md` pour les instructions d'installation de Qartez et @knip/mcp.

---

_Généré par quality-team v1 — `<timestamp ISO>`_
_Agents : scout · principles-auditor · refactor-executor · doc-updater_

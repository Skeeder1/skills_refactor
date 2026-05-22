# Rapport de refactoring — <date>

> Template REFACTOR_REPORT.md — utilisé par doc-updater.

## Résumé exécutif

| Métrique | Valeur |
|----------|--------|
| Scope analysé | `<scope>` |
| Mode | `<audit-only | refactor | all>` |
| Langages détectés | `<liste>` |
| Playbooks chargés | `<liste | aucun>` |
| Fichiers analysés | `<N>` |
| Violations bloquantes | `<N>` trouvées · `<N>` corrigées · `<N>` skipped |
| Violations importantes | `<N>` trouvées · `<N>` corrigées · `<N>` skipped |
| AI smells détectés | `<N>` |
| Code mort supprimé | `<N>` symboles |
| Plan validé | `<oui | non | audit-only>` |

## Validations

| Nom | Commande | Avant | Après | Sortie |
|-----|----------|-------|-------|--------|
| `<name>` | `<command>` | `<pass|fail|skipped>` | `<pass|fail|skipped>` | `<path>` |

_Si aucune validation n'est disponible : "Aucune commande de validation détectée pour ce projet."_

## Corrections appliquées

| Sévérité | Fichier | Changement | Type | Validé |
|----------|---------|------------|------|--------|
| `<blocking|important>` | `<path>` | `<description>` | `<type>` | `<oui|non>` |

_Si aucune correction n'a été appliquée, expliquer pourquoi : audit-only, plan non validé, validations absentes, manual_verify, ou safety gate._

## Éléments non traités

| Fichier | Raison | Recommandation |
|---------|--------|----------------|
| `<path>` | `<manual-verify|blacklisted|...>` | `<action>` |

## AI smells identifiés

| Fichier | Pattern | Description | Sévérité | Corrigé |
|---------|---------|-------------|----------|---------|
| `<path>` | `<pattern>` | `<description>` | `<severity>` | `<oui|non>` |

## Documentation mise à jour

- Documentation projet : `<oui|non>` — `<raison>`
- Références de chemins : `<oui|non>` — `<raison>`
- Documentation d'API publique : `<oui|non|non applicable>` — `<raison>`

## Prochaines étapes

1. Traiter les fichiers `manual_verify` après avoir ajouté le contexte ou les tests nécessaires.
2. Configurer les outils de validation manquants si le rapport indique `skipped`.
3. Relancer `/quality-team <scope>` après les corrections manuelles.

---

Généré par quality-team — `<timestamp ISO>`

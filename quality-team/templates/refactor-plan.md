# Plan d'action quality-team

## Scope

- Scope analysé : {{scope}}
- Mode demandé : {{mode}}
- Langages détectés : {{languages}}
- Playbooks chargés : {{playbooks}}

## Validations prévues

| Nom | Commande | Statut baseline | Raison |
|-----|----------|-----------------|--------|
| {{validation_name}} | `{{validation_command}}` | {{baseline_status}} | {{reason}} |

Si aucune commande n'est détectée, indiquer : validation automatique indisponible.

## Changements proposés

| Fichier | Sévérité | Problème | Changement prévu | Risque | Validation |
|---------|----------|----------|------------------|--------|------------|
| {{file}} | {{severity}} | {{problem}} | {{planned_change}} | {{risk}} | {{validation}} |

## Fichiers non touchés

| Fichier | Raison |
|---------|--------|
| {{file}} | {{reason}} |

## Limites

- Changements hors scope exclus.
- Fichiers générés, vendored, lockfiles et `DO NOT EDIT` exclus.
- Fichiers `manual_verify` exclus jusqu'à validation humaine séparée.
- Aucun refactor ne démarre sans validation explicite de l'utilisateur.

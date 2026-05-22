# Safe refactor rules
# Utilisé par : refactor-executor
# Règles universelles, indépendantes du langage et du framework.
---

Le refactor automatique doit être conservateur. Les playbooks peuvent ajouter des
cas spécifiques, mais seulement pour un stack détecté et seulement si le plan les
liste explicitement.

## Toujours sûr après confirmation outillage

Ces changements restent locaux et ne doivent pas modifier le comportement public :

- Supprimer du code mort confirmé par une analyse cross-fichiers ou Qartez.
- Supprimer des logs de debug non fonctionnels, hors audit/sécurité/observabilité.
- Supprimer des blocs de code commenté de plus de 3 lignes.
- Extraire une constante nommée dans le même fichier.
- Corriger la documentation publique sans changer signature ni logique.

## Sûr avec validation post-modification

Ces opérations nécessitent les validations détectées dans
`.claude/quality-team/validation_commands.json` :

- Renommer un symbole et mettre à jour tous les usages confirmés.
- Extraire une petite fonction locale sans changer l'interface publique.
- Déplacer une fonction interne vers un module voisin en mettant à jour tous les imports.
- Remplacer une valeur magique par une constante déjà introduite localement.
- Simplifier une condition ou un garde sans changer les cas couverts.

Si aucune validation projet n'est disponible, ces changements peuvent être proposés
dans le plan mais doivent être traités comme risque plus élevé.

## Jamais sans validation humaine séparée

- Changer une signature publique.
- Modifier un type ou format de retour public.
- Supprimer un fichier entier.
- Modifier authentification, autorisation, chiffrement, secrets, sessions ou permissions.
- Modifier migrations, schémas de base de données ou formats persistés.
- Modifier les tests au-delà d'un import/chemin nécessaire à un rename.
- Modifier la configuration de build, packaging, CI ou déploiement.
- Introduire une nouvelle dépendance.
- Changer une API externe ou un protocole réseau.

## Jamais toucher

- Fichiers générés ou vendored.
- Répertoires de build/cache (`dist/`, `build/`, `target/`, `.next/`, `out/`,
  `.cache/`, `vendor/`, équivalents).
- Lockfiles.
- Fichiers marqués `DO NOT EDIT` ou `GENERATED`.
- Fichiers listés dans `violations.manual_verify`.

## Protocole par fichier

1. Vérifier la blacklist et `manual_verify`.
2. Vérifier le blast radius avec Qartez pour les hotspots si disponible.
3. Lire le fichier et identifier le plus petit changement suffisant.
4. Appliquer uniquement ce qui est listé dans `refactor_plan.md`.
5. Lancer les validations applicables détectées.
6. Si une validation échoue, revert uniquement le fichier touché et logguer le skip.
7. Continuer avec le fichier suivant.

## Raisons de skip standard

- `manual-verify`
- `blacklisted`
- `blast-radius-too-high`
- `not-in-approved-plan`
- `missing-safe-refactor-rules`
- `validation-unavailable`
- `reverted:validation-failed`

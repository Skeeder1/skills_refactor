# AI smells — patterns de code générés par IA
# Source: synthétique
# Utilisé par : principles-auditor
# Référence universelle, indépendante du stack.
---

Ces patterns décrivent des défauts fréquents du code généré ou fortement assisté
par IA. Les exemples propres à un langage ou framework doivent vivre dans les
playbooks optionnels.

## 1. Over-generation

**Description :** Code, helpers, abstractions ou couches créés sans besoin réel.
**Signes :** Fonctions inutilisées, wrappers qui délèguent seulement, options jamais lues.
**Fix :** Supprimer le code confirmé inutilisé ou inliner les helpers triviaux.

## 2. Structural clones

**Description :** Même structure logique copiée dans plusieurs endroits.
**Signes :** Séquences validation → transformation → sauvegarde répétées.
**Fix :** Extraire seulement si le concept est réellement commun.

## 3. God modules

**Description :** Un fichier concentre plusieurs responsabilités.
**Signes :** Beaucoup d'imports, exports hétérogènes, forte complexité, churn élevé.
**Fix :** Séparer orchestration, domaine, I/O et présentation.

## 4. Swallowed errors

**Description :** Erreurs capturées, ignorées ou remplacées par une valeur neutre.
**Signes :** Catch vide, résultat critique ignoré, fallback silencieux.
**Fix :** Propager, mapper ou traiter explicitement.

## 5. Magic values

**Description :** Nombres ou chaînes métier sans nom.
**Signes :** Seuils, délais, statuts ou clés hardcodés sans constante.
**Fix :** Extraire une constante nommée proche de son usage.

## 6. Dual source of truth

**Description :** Même information maintenue dans plusieurs endroits.
**Signes :** Cache, état dérivé ou config dupliqués sans règle d'invalidation.
**Fix :** Choisir une source autoritaire et dériver le reste.

## 7. Speculative state drift

**Description :** État local modifié avant confirmation durable, sans rollback.
**Signes :** Écriture optimiste ou cache mis à jour sans réconciliation.
**Fix :** Confirmer, rollback ou recharger depuis la source autoritaire.

## 8. Stale context capture

**Description :** Callback, closure ou job asynchrone utilise un contexte périmé.
**Signes :** Valeur capturée avant changement de config, session ou état.
**Fix :** Passer explicitement les valeurs nécessaires ou réévaluer au moment d'agir.

## 9. Direct shared-state mutation

**Description :** Mutation d'un objet partagé sans passer par son propriétaire.
**Signes :** Modification en place d'une structure utilisée par plusieurs consommateurs.
**Fix :** Centraliser la mutation et préserver les invariants.

## 10. Lifecycle dependency drift

**Description :** Initialisation, refresh ou teardown dépend d'un ordre implicite.
**Signes :** Ressource créée sans chemin clair de mise à jour ou destruction.
**Fix :** Rendre le cycle de vie explicite et testé.

## 11. Unvalidated external data

**Description :** Données externes utilisées comme données internes fiables.
**Signes :** Parsing direct, absence de validation, hypothèses de format implicites.
**Fix :** Valider et normaliser à la frontière.

## 12. Type or contract escape hatch

**Description :** Contournement du système de contrats du langage.
**Signes :** Cast large, dynamic escape hatch, suppression de warning sans justification.
**Fix :** Narrowing, validation ou type domaine explicite.

## 13. Missing cleanup

**Description :** Ressource ouverte sans libération claire.
**Signes :** Listener, timer, fichier, connexion, transaction ou lock sans cleanup.
**Fix :** Ajouter teardown, scope management ou finally/defer équivalent.

## 14. Pass-through plumbing

**Description :** Paramètres passés à travers plusieurs couches sans valeur ajoutée.
**Signes :** Chaînes d'appels qui ne font que relayer les mêmes arguments.
**Fix :** Revoir la frontière ou rapprocher le consommateur de la source.

## 15. Export surface explosion

**Description :** Trop d'éléments exposés comme API publique.
**Signes :** Exports globaux, re-exports massifs, modules sans frontière claire.
**Fix :** Exposer seulement l'API intentionnelle.

## 16. Premature abstraction

**Description :** Interfaces, factories ou bases abstraites avant plusieurs usages réels.
**Signes :** Une seule implémentation, aucun bénéfice de substitution.
**Fix :** Revenir au concret jusqu'à besoin prouvé.

## 17. Error type erasure

**Description :** Conversion d'erreurs riches en messages génériques.
**Signes :** Perte de cause, code, contexte ou action corrective.
**Fix :** Préserver la cause et mapper vers une erreur domaine.

## 18. Unit doing too much

**Description :** Une fonction, classe ou module orchestre trop de phases.
**Signes :** Lecture, validation, transformation, persistance et rendu ensemble.
**Fix :** Extraire les phases et nommer les responsabilités.

## 19. Stringly-typed protocols

**Description :** Protocoles internes basés sur chaînes libres.
**Signes :** Noms d'événements, commandes ou statuts répétés sans contrat.
**Fix :** Centraliser les constantes ou introduire un type/enum/schema.

## 20. Async without failure policy

**Description :** Opération asynchrone sans stratégie d'échec.
**Signes :** Pas de timeout, retry, propagation, cancellation ou fallback explicite.
**Fix :** Définir la politique d'erreur et de durée.

## 21. Redundant derived data

**Description :** Valeur dérivée stockée et synchronisée manuellement.
**Signes :** Totaux, listes filtrées ou statuts recalculables persistés localement.
**Fix :** Calculer à la demande ou documenter l'invalidation.

## 22. Assertion on external data

**Description :** Le code affirme un format externe au lieu de le vérifier.
**Signes :** Cast/deserialize permissif sans validation de domaine.
**Fix :** Parser strictement à la frontière.

## 23. Missing loading, empty or error states

**Description :** Seul le happy path est représenté.
**Signes :** Absence de comportement pour vide, erreur, timeout ou permission refusée.
**Fix :** Traiter les états attendus du workflow.

## 24. Hidden side effects

**Description :** Effet de bord dans une fonction qui semble pure.
**Signes :** Nom de lecture ou formatage qui écrit, réseau, loggue ou modifie un global.
**Fix :** Séparer ou renommer explicitement.

## 25. Overly generic naming

**Description :** Noms vagues dans du code non trivial.
**Signes :** `data`, `result`, `handler`, `manager`, `process`, `doStuff`.
**Fix :** Nommer selon le domaine et l'action réelle.

## 26. Debug artifact proliferation

**Description :** Traces de debug laissées dans le code final.
**Signes :** Logs temporaires, prints, dumps, TODO sans propriétaire.
**Fix :** Supprimer ou remplacer par instrumentation structurée justifiée.

## 27. Missing public contract documentation

**Description :** API publique sans contrat lisible.
**Signes :** Exports, commandes, endpoints ou modules publics sans description des entrées/sorties/erreurs.
**Fix :** Documenter dans le format idiomatique du langage.

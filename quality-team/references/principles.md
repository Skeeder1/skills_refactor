# Principes fondamentaux — quality-team
# Utilisé par : principles-auditor
# Référence universelle, indépendante du langage et du framework.
---

Ces principes s'appliquent à tout codebase. Les détails propres à un langage,
framework ou runtime doivent vivre dans un playbook optionnel, jamais dans ce
fichier.

## P1 — Responsabilité unique

**Règle :** Un fichier, module, type ou fonction doit avoir une raison principale
de changer.

**Violations détectables :**
- Module mélangeant orchestration, accès aux données, rendu, validation et I/O.
- Fonction qui fait setup, validation, calcul, persistance et reporting.
- Fichier très long avec plusieurs domaines métier ou techniques non liés.

**Fix standard :**
- Extraire les responsabilités en modules ou fonctions nommées.
- Séparer orchestration, règles métier, accès externe et présentation.
- Garder les interfaces publiques petites et intentionnelles.

**Détection automatique :**
- Hotspots Qartez, complexité Lizard, graphe de dépendances, nombre d'exports.

## P2 — Source unique de vérité

**Règle :** Une information métier ou technique doit avoir une source autoritaire.
Les copies, caches et vues dérivées doivent être explicitement synchronisés ou
recalculables.

**Violations détectables :**
- Même valeur maintenue dans deux structures sans source autoritaire claire.
- Cache mis à jour manuellement sans invalidation.
- État dérivé stocké au lieu d'être calculé.
- Configuration dupliquée entre code, fichier et environnement.

**Fix standard :**
- Choisir la source autoritaire.
- Dériver les vues secondaires.
- Centraliser la config et documenter les règles d'invalidation.

**Détection automatique :**
- Clones Qartez/Lizard, duplication de constantes, co-change fréquent.

## P3 — Contrats explicites aux frontières

**Règle :** Toute donnée venant d'une frontière externe doit être validée,
normalisée ou typée selon les conventions du langage.

**Frontières :**
- API, fichiers, base de données, CLI, variables d'environnement, réseau,
  messages inter-processus, sérialisation, entrée utilisateur.

**Violations détectables :**
- Donnée externe utilisée directement comme donnée interne fiable.
- Cast/assertion sans validation runtime lorsque le langage le nécessite.
- Parsing sans gestion d'erreur.
- Format implicite non documenté.

**Fix standard :**
- Introduire un parseur, validateur, DTO ou type domaine.
- Refuser ou normaliser les données invalides à la frontière.
- Documenter le contrat public.

**Détection automatique :**
- Linters, typecheckers, grep sur parse/cast, règles sécurité, tests de contrat.

## P4 — Intégrité des mutations et invariants

**Règle :** Toute mutation doit préserver les invariants du domaine et rester
localisée. Les états partagés ou observables doivent être modifiés de manière
contrôlée.

**Violations détectables :**
- Mutation directe d'un objet partagé sans passer par l'API propriétaire.
- Mise à jour partielle qui laisse un invariant cassé en cas d'erreur.
- Donnée modifiée en place alors que les consommateurs attendent une nouvelle valeur.

**Fix standard :**
- Centraliser les mutations dans une fonction ou méthode domaine.
- Utiliser des transactions, copies contrôlées ou garde-fous selon le langage.
- Valider les invariants avant et après mutation critique.

**Détection automatique :**
- Linters, analyse AST, tests d'invariants, revue des hotspots.

## P5 — Erreurs explicites

**Règle :** Une erreur doit être traitée, propagée ou convertie en erreur domaine.
Elle ne doit jamais disparaître silencieusement.

**Violations détectables :**
- Catch vide ou qui retourne une valeur neutre sans justification.
- Résultat d'opération critique ignoré.
- Exception/panic/abort possible sur entrée utilisateur.
- Message d'erreur générique qui perd la cause utile.

**Fix standard :**
- Propager l'erreur ou la mapper vers un type/format domaine.
- Logger seulement si le log est actionnable et ne remplace pas la propagation.
- Ajouter une stratégie de fallback explicite pour les erreurs acceptées.

**Détection automatique :**
- Linters, analyse des blocs catch/match/result, tests d'échec.

## P6 — Effets de bord isolés

**Règle :** Les effets de bord doivent être explicites, localisés et séparés du
calcul pur quand c'est raisonnable.

**Violations détectables :**
- Fonction de calcul qui écrit un fichier, modifie un global ou déclenche du réseau.
- I/O cachée dans un getter, formatteur ou validateur.
- Ordre d'exécution implicite nécessaire au bon fonctionnement.

**Fix standard :**
- Séparer calcul pur et orchestration.
- Injecter les dépendances externes.
- Nommer les fonctions à effet avec des verbes explicites.

**Détection automatique :**
- Appels I/O dans fonctions de transformation, graphes d'appels, revue des noms.

## P7 — Duplication maîtrisée

**Règle :** Deux occurrences similaires sont un signal, trois occurrences
indiquent souvent une extraction. L'abstraction ne doit pas créer plus de
couplage que la duplication.

**Violations détectables :**
- Même séquence validation → transformation → sauvegarde dans plusieurs fichiers.
- Constantes ou messages métier copiés.
- Tests ou handlers dupliquant la même logique.

**Fix standard :**
- Extraire une fonction, un module ou un type commun si le concept est réellement partagé.
- Garder séparé si les domaines divergent.

**Détection automatique :**
- `qartez_clones`, Lizard duplicate, recherche de constantes.

## P8 — Nommage intentionnel

**Règle :** Les noms doivent exposer l'intention métier ou technique, pas seulement
la forme de la donnée.

**Violations détectables :**
- Noms vagues dans du code non trivial : `data`, `value`, `result`, `tmp`, `manager`.
- Même concept nommé différemment selon les modules.
- Fonction nommée comme un événement alors qu'elle transforme ou persiste.

**Fix standard :**
- Renommer selon le vocabulaire du domaine.
- Utiliser un terme unique par concept.
- Préférer des verbes précis pour les actions.

**Détection automatique :**
- Recherche de noms génériques, revue de cohérence, refs Qartez avant rename.

## P9 — Petite taille et faible complexité

**Règle :** Les unités de code doivent rester lisibles, testables et peu
imbriquées.

**Seuils par défaut :**
- Fonction > 40 lignes : à inspecter.
- Complexité cyclomatique > 10 : à inspecter.
- Paramètres > 4 : envisager un objet/struct de paramètres.
- Imbrication > 3 niveaux : préférer gardes ou extraction.

**Fix standard :**
- Extraire les phases.
- Remplacer l'imbrication par des gardes.
- Introduire un objet de paramètres si cela clarifie l'appel.

**Détection automatique :**
- Lizard, Qartez hotspots, linters de complexité.

## P10 — Documentation vivante

**Règle :** La documentation doit expliquer les contrats, décisions et contraintes
qui ne sont pas évidents dans le code.

**Violations détectables :**
- API publique sans contrat compréhensible.
- README ou AGENTS qui référence des chemins obsolètes.
- Commentaire qui décrit le code au lieu d'expliquer le pourquoi.
- TODO sans contexte, propriétaire ou condition de résolution.

**Fix standard :**
- Documenter les APIs publiques dans le format idiomatique du langage.
- Mettre à jour la documentation quand un module bouge.
- Supprimer les commentaires narratifs inutiles.

**Détection automatique :**
- Recherche d'exports publics non documentés, liens cassés, TODO sans contexte.

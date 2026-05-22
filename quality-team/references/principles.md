# Principes fondamentaux — quality-refactor-team
# Utilisé par : principles-auditor
# Ces 10 principes sont la référence pour classifier les violations dans violations.json
---

## P1 — Single Responsibility (Responsabilité unique)

**Règle :** Un fichier, une fonction, un module = une seule raison de changer.
Un composant React ne doit pas gérer à la fois la data fetching, la logique métier et le rendu.
Une fonction ne doit pas mixer setup, validation, computation et side effects.

**Violations détectables :**
- Fichier > 300 lignes avec des imports de domaines différents (UI + store + réseau)
- Fonction qui retourne une valeur ET produit un side effect (mutation d'état + calcul)
- Hook React qui fait plus d'une chose (fetch + transform + format)
- Composant avec `useState` pour 5+ valeurs non liées

**Fix standard :**
- Découper par responsabilité : state → store/hook, fetch → service, rendu → composant
- Extraire les phases : validation → `validateX()`, transformation → `transformX()`, save → `saveX()`
- Séparer commandes (mutations) et queries (lectures)

**Détection automatique :**
- `qartez_hotspots` (score élevé = fort couplage + complexité)
- `lizard` : NLOC > 300, CCN > 10
- `qartez_deps` : fichier avec > 10 dépendances directes

---

## P2 — Single Source of Truth (Source unique de vérité)

**Règle :** Un même état ne doit exister qu'en un seul endroit autoritaire.
Jamais de synchronisation manuelle entre deux stores ou entre le frontend et le backend.

**Violations détectables :**
- État dupliqué entre Zustand et `useState` local
- State Tauri/IPC mis à jour optimistement sans reconciliation depuis le backend Rust
- Variable locale qui "miroir" un champ de store sans mécanisme de sync
- Même valeur calculée dans deux endroits différents (deux `useMemo` identiques)

**Fix standard :**
- Dériver l'état affiché depuis une seule source : `const total = useStore(s => s.items.length)`
- Après une mutation IPC, toujours refetch ou utiliser la réponse du backend comme nouvelle source
- Supprimer les `useState` qui dupliquent des champs de store

**Détection automatique :**
- `qartez_clones` (logique de state dupliquée)
- Grep : `useState.*store\.|store\..*useState` dans le même fichier

---

## P3 — Immutabilité dans les contextes réactifs

**Règle :** Ne jamais muter directement un état React, un store Zustand (sans Immer),
ou une prop passée en paramètre. Toujours retourner un nouvel objet/tableau.

**Violations détectables :**
- `state.items.push(item)` sur du state React
- `obj.field = value` sur un objet reçu en prop
- `array[0] = newValue` dans un setter Zustand sans Immer
- `Object.assign(state, newState)` sur le state directement

**Fix standard :**
- `setItems(prev => [...prev, item])`
- Retourner un nouvel objet : `{ ...state, field: value }`
- Utiliser Immer dans Zustand pour les mutations complexes

**Détection automatique :**
- `biome: noDirectMutation`
- Grep : `\.push\(|\.splice\(|\.shift\(` dans des contextes de state React

---

## P4 — Contrats typés aux frontières

**Règle :** Toute donnée qui entre dans le système depuis l'extérieur (API REST, IPC Tauri,
localStorage, URL params, env vars, formulaires) doit être validée et typée explicitement.
Jamais de `as SomeType` sur des données dynamiques sans validation.

**Violations détectables :**
- `const data = await fetch(url).then(r => r.json())` sans schema Zod/Valibot
- `JSON.parse(localStorage.getItem('x'))` sans try/catch ni schema
- `as User` ou `as ApiResponse` sur une réponse externe sans parse
- `any` sur des paramètres de fonctions d'API publique
- `tsconfig.json` sans `strict: true`
- Commande Tauri `#[tauri::command]` sans retour `Result<T, E>`

**Fix standard :**
- Zod : `const UserSchema = z.object({...}); const user = UserSchema.parse(data)`
- TypeScript strict : `"strict": true` dans tsconfig
- Rust : toutes les commandes Tauri retournent `Result<T, String>` ou `Result<T, AppError>`

**Détection automatique :**
- `biome` : no-explicit-any
- `tsc --strict`
- Grep : `as [A-Z][a-zA-Z]+\b` sur des résultats de fetch/parse

---

## P5 — Erreurs explicites (pas de continuation silencieuse)

**Règle :** Une erreur doit toujours soit être propagée, soit être traitée explicitement
avec un type d'erreur domain. Jamais de `catch {}` vide, jamais d'erreur avalée.

**Violations détectables :**
- `catch (e) {}` vide
- `catch (e) { console.log(e) }` sans propagation
- `.catch(() => null)` ou `.catch(() => undefined)` sur des opérations critiques
- `let _ = result_qui_peut_echouer` en Rust
- `unwrap()` / `expect()` sur des chemins utilisateur en Rust
- Fonctions async sans aucune gestion d'erreur

**Fix standard :**
- Propager : `catch (e) { logger.error(e); throw e }`
- Retourner un Result : `type Result<T> = { ok: true; data: T } | { ok: false; error: string }`
- Rust : `result?` ou `result.map_err(|e| AppError::from(e))`

**Détection automatique :**
- `clippy::unwrap_used`, `clippy::expect_used`
- `biome` : no-console (pour détecter les catch avec seulement console)
- Grep : `catch\s*\(\w+\)\s*\{\s*\}` (catch vide)

---

## P6 — UI pure (render = f(state))

**Règle :** Le rendu d'un composant React doit être une fonction pure de son state et ses props.
Pas de side effects dans le corps du composant. Pas d'appels réseau en dehors de `useEffect`.
Pas de mutation de refs pendant le rendu.

**Violations détectables :**
- Appel de fonction avec side effect dans le corps du composant (hors handlers et useEffect)
- `fetch()` appelé directement dans le rendu sans `useEffect`
- Mutation d'une ref ou d'une variable externe pendant le rendu
- Logique conditionnelle qui change l'ordre des hooks (hooks dans des conditions)

**Fix standard :**
- Déplacer tout side effect dans `useEffect` ou un event handler
- Dériver les valeurs calculées avec `useMemo` plutôt que de les calculer pendant le rendu

**Détection automatique :**
- `eslint-plugin-react-hooks` : rules-of-hooks
- `biome` : lint/correctness/useHookAtTopLevel

---

## P7 — DRY avec seuil pragmatique

**Règle :** 
- 2 occurrences identiques → surveiller
- 3 occurrences identiques → extraire obligatoirement
- Ne pas extraire si l'abstraction crée plus de couplage que de valeur

**Violations détectables :**
- 3+ fonctions avec la même structure interne (même try/catch, même validation)
- 3+ composants avec le même `useEffect` copié-collé
- 3+ fichiers avec le même bloc de formatage de date ou de validation

**Fix standard :**
- Extraire en fonction partagée dans `utils/` ou hook custom
- Ne pas abstraire si les 3 occurrences sont dans des domaines différents (couplage artificiel)

**Détection automatique :**
- `qartez_clones` (duplication AST)
- `lizard --duplicate`

---

## P8 — Nommage intentionnel

**Règle :**
- Fonctions : verbes d'action (`fetchUser`, `validateEmail`, `formatDate`)
- Booléens : préfixe `is`, `has`, `can`, `should` (`isLoading`, `hasError`)
- Pas de noms génériques : `data`, `result`, `item`, `handler`, `doStuff`, `process`
- Un terme par concept dans tout le codebase (`user` ou `account`, pas les deux)

**Violations détectables :**
- Variables nommées `data`, `result`, `temp`, `val`, `x`, `obj` dans des fonctions longues
- Fonctions nommées `handleData`, `processItem`, `manageState`
- Booléens nommés `loaded`, `error` (sans préfixe `is`/`has`)
- Même concept nommé différemment selon les fichiers

**Fix standard :**
- Renommer avec des noms qui expriment l'intention métier
- Choisir un vocabulaire commun et l'appliquer dans tout le codebase

**Détection automatique :**
- `qartez_grep` : rechercher les noms génériques connus
- Revue manuelle pour la cohérence du vocabulaire

---

## P9 — Fonctions petites et lisibles

**Règle :**
- ≤ 40 lignes par fonction
- ≤ 3 niveaux d'imbrication (if dans for dans if = déjà problématique)
- ≤ 4 paramètres (sinon utiliser un objet de configuration)
- Early return pour les cas d'erreur (pas de `else` après un `return` ou `throw`)

**Violations détectables :**
- Fonctions > 40 lignes (NLOC)
- Complexité cyclomatique > 10 (CCN)
- > 4 paramètres de fonction
- > 3 niveaux d'imbrication (calculé par lizard)

**Fix standard :**
- Extraire les phases : garde → validation → traitement → retour
- Early return : `if (!isValid) return null` en tête de fonction
- Grouper les paramètres en un objet typé : `function fn(opts: { a: string; b: number })`

**Détection automatique :**
- `lizard` : `--CCN 10 --length 40 --arguments 4`
- `qartez_hotspots` (CCN élevé)

---

## P10 — Documentation vivante

**Règle :**
- JSDoc sur toutes les fonctions exportées (au minimum `@param` et `@returns`)
- `AGENTS.md` à jour quand un module est ajouté, déplacé ou supprimé
- Les commentaires expliquent le POURQUOI (contrainte, décision), jamais le QUOI
- Pas de commentaires `// TODO` sans ticket ou date associée

**Violations détectables :**
- Fonction exportée sans JSDoc
- Paramètres de fonction sans type annoté ET sans JSDoc
- `// TODO` sans référence (ticket, issue, date)
- `AGENTS.md` qui liste des modules qui n'existent plus
- Commentaires qui narrent le code (`// get the user and return it`)

**Fix standard :**
- Ajouter JSDoc minimal : `/** @param userId {string} @returns {Promise<User>} */`
- Supprimer ou convertir les commentaires narratifs
- Mettre à jour AGENTS.md après chaque changement de structure

**Détection automatique :**
- `biome` (jsdoc lint rules si configuré)
- `qartez_outline` (surfaces les fonctions exportées sans doc)
- Grep : `^export (function|const|class)` sans ligne `\*\*` précédente

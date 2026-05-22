# AI smells — patterns de code générés par IA
# Source: synthétique — URL inaccessible (voidborne-d/sober-coding 404)
# Utilisé par : principles-auditor
---

Les 27 patterns ci-dessous sont des anti-patterns fréquemment introduits par les LLMs
lors de la génération de code. Le principles-auditor les détecte dans les fichiers flaggés
par le scout et les classe selon leur sévérité.

---

## 1. Over-generation (Verbosité excessive)

**Description :** Fonctions, classes ou modules créés en excès par rapport au besoin réel.
L'IA génère souvent plus de code que nécessaire "au cas où".

**Signes :** Fonctions helper inutilisées à côté de la fonction principale. Abstractions
vides qui ne font que déléguer. Fichiers `utils.ts` de 800 lignes pour 3 usages.

**Fix :** Supprimer tout code sans importeur confirmé. Inliner les helpers triviaux (<3 lignes).

---

## 2. Structural clones (Duplicats structurels)

**Description :** Blocs de logique copiés quasi-identiquement dans plusieurs fichiers ou
composants. L'IA répète les patterns plutôt que de les abstraire.

**Signes :** 3+ fonctions avec la même structure `try/catch → validate → transform → return`.
Composants React avec le même `useEffect` copié-collé.

**Détection :** `qartez_clones`, `lizard --duplicate`

**Fix :** Extraire en fonction partagée ou hook custom.

---

## 3. God files (Fichiers dieu)

**Description :** Un seul fichier concentre trop de responsabilités : state, UI, logique
métier, appels réseau, validation. L'IA "complète" le fichier existant sans le découper.

**Signes :** Fichier > 400 lignes avec des imports de 10+ modules différents. Un seul
fichier exportant à la fois des composants, des stores et des utils.

**Détection :** `qartez_hotspots` (score élevé), `lizard` (NLOC > 400)

**Fix :** Découper par responsabilité : state → store, UI → component, logique → hook/service.

---

## 4. Swallowed errors (Erreurs avalées)

**Description :** Les erreurs sont capturées mais ignorées silencieusement. L'IA génère
des `try/catch` "défensifs" qui cachent les vrais problèmes.

**Signes :** `catch (e) {}` vide. `catch (e) { console.log(e) }` sans re-throw ni propagation.
`.catch(() => null)` sur des Promises critiques. `let _ = result_qui_peut_echouer` en Rust.

**Fix :** Toujours soit logger+propager, soit retourner un type `Result<T, E>` / `Either`.

---

## 5. Magic values (Valeurs magiques)

**Description :** Constantes numériques ou string codées en dur sans nom explicatif.

**Signes :** `if (status === 3)`, `setTimeout(fn, 5000)`, `slice(0, 50)` sans constante nommée.

**Fix :** Extraire en constante nommée dans le même fichier : `const MAX_RETRIES = 3`.

---

## 6. Dual state sources (Double source d'état)

**Description :** Le même état existe simultanément dans deux endroits qui peuvent diverger :
store Zustand + local `useState`, state Redux + variable locale, store + backend Rust non resynchronisé.

**Signes :** `const [items, setItems] = useState(store.items)` suivi d'une mutation locale
sans resync. State Tauri IPC mis à jour optimistement sans reconciliation depuis le backend.

**Fix :** Single source of truth. Dériver l'état affiché depuis une seule source autoritaire.

---

## 7. Optimistic UI drift (Dérive UI optimiste)

**Description :** L'UI est mise à jour avant la confirmation du backend, et n'est jamais
resynchronisée en cas d'échec ou de désaccord.

**Signes :** `setItems([...items, newItem])` avant le `await invoke('add_item')`.
Pas de `.catch()` qui revert l'état.

**Fix :** Soit optimistic avec rollback sur erreur, soit pessimiste (UI update après confirmation).

---

## 8. Stale closures (Fermetures périmées)

**Description :** Un callback ou `useEffect` capture une valeur de state ou prop qui
devient obsolète après un re-render.

**Signes :** `useEffect(() => { doSomething(value) }, [])` où `value` change.
Event listeners qui lisent des variables de state sans les avoir dans leurs dépendances.

**Détection :** `biome: useExhaustiveDependencies`, `eslint-plugin-react-hooks`

**Fix :** Ajouter la valeur aux dépendances, ou utiliser `useRef` pour les valeurs stables.

---

## 9. Direct mutation (Mutation directe)

**Description :** Mutation directe du state React ou d'un objet Zustand plutôt que
de retourner un nouvel objet immutable.

**Signes :** `state.items.push(item)`, `obj.field = value` sur un objet passé en props.
`array[0] = newValue` dans un setter Zustand sans Immer.

**Détection :** `biome: noDirectMutation`

**Fix :** `setItems([...items, item])`, retourner un nouveau state avec spread ou Immer.

---

## 10. useEffect deps incorrects (Dépendances useEffect erronées)

**Description :** Le tableau de dépendances de `useEffect` est vide `[]` alors que l'effet
utilise des valeurs qui changent, ou inversement contient trop de dépendances provoquant
des boucles infinies.

**Signes :** `useEffect(() => { fetchData(id) }, [])` où `id` est un prop.
`useEffect(..., [object])` avec un objet recréé à chaque render.

**Détection :** `biome: useExhaustiveDependencies`

**Fix :** Deps précises. Stabiliser les références avec `useMemo`/`useCallback` si nécessaire.

---

## 11. Unvalidated external data (Données externes non validées)

**Description :** Les données venant de l'API, de l'IPC Tauri, du localStorage ou de l'URL
sont utilisées directement sans validation ni parsing typé.

**Signes :** `const data = await fetch(url).then(r => r.json())` puis `data.field` sans check.
`JSON.parse(localStorage.getItem('x'))` sans try/catch ni schema.

**Fix :** Valider avec Zod, Valibot, ou `io-ts` à chaque frontière externe.

---

## 12. Implicit any proliferation (Propagation implicite d'any)

**Description :** L'IA utilise `any` ou laisse TypeScript inférer `any` sur des données
dynamiques, ce qui neutralise le typage en cascade.

**Signes :** `const data: any = ...`, `function handler(e: any)`, `as any` pour contourner
une erreur TS plutôt que de corriger le type.

**Fix :** Types précis ou `unknown` avec narrowing explicite. Jamais `as any` sur des frontières.

---

## 13. Missing cleanup in useEffect (Nettoyage useEffect absent)

**Description :** Des effets qui créent des abonnements, des timers, ou des event listeners
ne retournent pas de fonction de cleanup, causant des fuites mémoire.

**Signes :** `useEffect(() => { window.addEventListener('resize', fn) }, [])` sans `return () => window.removeEventListener(...)`.

**Fix :** Toujours retourner une fonction de cleanup pour les abonnements et timers.

---

## 14. Props drilling anti-pattern

**Description :** Des props sont passées à travers 4+ niveaux de composants sans utilité
intermédiaire, créant un couplage fort et des refactorings coûteux.

**Signes :** `<A user={user}><B user={user}><C user={user}><D user={user}/></C></B></A>`

**Fix :** Context React, Zustand store, ou composition de composants.

---

## 15. Barrel file explosion (Explosion de barrel files)

**Description :** Des `index.ts` qui réexportent tout (`export * from './...'`) dans chaque
dossier, créant des dépendances circulaires et des bundles surchargés.

**Signes :** `index.ts` avec 20+ `export *`. Imports circulaires détectés par knip ou webpack.

**Fix :** Exports nommés explicites. Ne pas réexporter tout ce qui n'est pas une API publique.

---

## 16. Premature abstraction (Abstraction prématurée)

**Description :** L'IA crée des interfaces, factories, et classes abstraites avant qu'il
y ait 2+ implémentations concrètes, pour "anticiper l'extensibilité".

**Signes :** `interface IUserRepository` avec une seule implémentation `UserRepository`.
`abstract class BaseHandler` pour une seule sous-classe.

**Fix :** Concrétiser d'abord. Abstraire seulement à la 2ème ou 3ème implémentation.

---

## 17. Error type erasure (Effacement du type d'erreur)

**Description :** Les erreurs typées sont capturées comme `unknown` ou `Error` générique
puis relancées sans type, perdant l'information structurée.

**Signes :** `catch (e: unknown) { throw new Error(String(e)) }`. Rust : `map_err(|_| "error")`.

**Fix :** Preserv le type d'erreur original ou mapper vers un type d'erreur domaine précis.

---

## 18. Component doing too much (Composant trop chargé)

**Description :** Un composant React gère à la fois la data fetching, la logique métier,
la transformation des données ET le rendu. Violation directe de P1 (SRP).

**Signes :** Composant avec `useEffect` pour fetch + `useState` pour 5+ valeurs + 100+ lignes de JSX.

**Fix :** Extraire la logique dans un custom hook. Séparer les conteneurs des composants de présentation.

---

## 19. String-based event bus (Bus d'événements par string)

**Description :** Communication entre modules via des strings hardcodées comme noms d'événements,
sans typage ni contrat.

**Signes :** `eventBus.emit('user:updated', data)` et `eventBus.on('user:updated', handler)`
dans des fichiers différents sans type commun.

**Fix :** Typer les événements avec une union discriminée ou un enum.

---

## 20. Async without error boundary (Async sans gestion d'erreur)

**Description :** Des fonctions async ne gèrent pas les cas d'erreur, laissant des Promises
rejetées non catchées qui crashent silencieusement.

**Signes :** `async function load() { const data = await fetch(url); setData(data.json()) }`
sans try/catch ni `.catch()`.

**Fix :** Toujours wrapper les appels async critiques dans try/catch ou utiliser Result types.

---

## 21. Redundant state (État redondant)

**Description :** État dérivé stocké dans un `useState` séparé alors qu'il pourrait être
calculé directement depuis d'autres états.

**Signes :** `const [filteredItems, setFilteredItems] = useState([])` mis à jour dans un
`useEffect` qui dépend de `items` et `filter`.

**Fix :** Calculer directement : `const filteredItems = useMemo(() => items.filter(...), [items, filter])`.

---

## 22. Type assertion on external data (Assertion de type sur données externes)

**Description :** Utilisation de `as SomeType` sur des données venant de l'extérieur
(API, IPC, JSON) sans validation réelle.

**Signes :** `const user = JSON.parse(data) as User`. `const result = await invoke(...) as ApiResponse`.

**Fix :** Parser et valider avec Zod. L'assertion de type ne valide pas à runtime.

---

## 23. Missing loading/error states (États loading/error absents)

**Description :** L'IA génère le happy path mais oublie de gérer les états intermédiaires
(loading, error, empty) dans les composants qui font des appels async.

**Signes :** Composant qui affiche `data.map(...)` sans vérifier si `data` est défini
ou si le fetch a échoué.

**Fix :** Toujours gérer 3 états : loading, error, success.

---

## 24. Implicit side effects in render (Effets de bord dans le render)

**Description :** Des side effects (mutation de state, appels réseau, accès au DOM) sont
déclenchés directement dans le corps d'un composant React, pas dans un `useEffect`.

**Signes :** `function Component() { fetchData(); return <div/> }` — `fetchData` appelé
à chaque render.

**Fix :** Tout side effect doit être dans `useEffect`, un event handler, ou un hook.

---

## 25. Overly generic naming (Nommage trop générique)

**Description :** L'IA nomme les variables et fonctions avec des termes vagues : `data`,
`result`, `handler`, `process`, `manage`, `handle`, `doStuff`.

**Signes :** `const data = await getData()`. `function handleClick(e) { processData(e.target.value) }`.

**Fix :** Noms qui expriment l'intention : `const userProfile = await fetchUserProfile(userId)`.

---

## 26. Console.log proliferation (Logs de debug en production)

**Description :** L'IA laisse des `console.log`, `console.error`, `print!`, `dbg!`
utilisés pendant le développement dans le code final.

**Signes :** `console.log('data:', data)` dans des fonctions de production. `dbg!(value)` en Rust.

**Fix :** Supprimer tous les logs de debug. Utiliser un logger structuré avec niveaux si des logs sont nécessaires.

---

## 27. Missing return type annotations (Annotations de retour absentes)

**Description :** Les fonctions exportées n'ont pas d'annotation de type de retour explicite,
forçant TypeScript à inférer un type complexe qui peut changer silencieusement.

**Signes :** `export function getUser(id: string) { ... }` sans `: Promise<User>`.
Inférence de `any` ou de types union complexes sur les fonctions d'API publique.

**Fix :** Annoter explicitement le type de retour de toute fonction exportée.

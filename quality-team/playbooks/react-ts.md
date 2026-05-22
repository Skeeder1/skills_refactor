# Playbook — React + TypeScript + Tauri
# Utilisé par : principles-auditor (chargé si le projet contient src-tauri/ ou .tsx)
# Référence croisée : principles.md (P1-P10), ai-smells.md
---

Ce playbook liste les violations et patterns spécifiques aux projets React + TypeScript + Tauri.
Il complète `principles.md` avec des règles concrètes liées à ce stack.

---

## Section 1 — State management

### 1.1 Mutation directe du state React → violation P3

**Pattern interdit :**
```typescript
// ❌ Mutation directe
state.items.push(newItem)
state.items.splice(index, 1)
items[0] = updatedItem
```

**Correction :**
```typescript
// ✅ Retour immutable
setItems(prev => [...prev, newItem])
setItems(prev => prev.filter((_, i) => i !== index))
setItems(prev => prev.map((item, i) => i === 0 ? updatedItem : item))
```

**Détection :** `biome: noDirectMutation` · Grep : `\.push\(|\.splice\(|\.shift\(` dans handlers React

---

### 1.2 État dupliqué entre Zustand et Tauri backend → violation P2

**Pattern interdit :**
```typescript
// ❌ Mise à jour optimiste sans reconciliation
const addEntry = async (entry: Entry) => {
  setEntries([...entries, entry])  // mise à jour locale optimiste
  await invoke('add_entry', { entry }) // pas de resync depuis la réponse
}
```

**Correction :**
```typescript
// ✅ Source unique depuis le backend
const addEntry = async (entry: Entry) => {
  const updatedEntries = await invoke<Entry[]>('add_entry', { entry })
  setEntries(updatedEntries)  // la réponse backend fait autorité
}
```

**Détection :** Grep : `invoke(` + `set[A-Z]` sans lecture du résultat de l'invoke dans la même fonction

---

### 1.3 useEffect avec deps incorrectes → violation P3

**Pattern interdit :**
```typescript
// ❌ Deps vides mais effet dépend de props/state
useEffect(() => {
  fetchData(userId) // userId change mais l'effet ne se relance pas
}, [])

// ❌ Deps trop larges causant une boucle infinie
useEffect(() => {
  process(config)
}, [config]) // si config est un objet recréé à chaque render
```

**Correction :**
```typescript
// ✅ Deps précises
useEffect(() => {
  fetchData(userId)
}, [userId])

// ✅ Stabiliser avec useMemo si config est un objet
const stableConfig = useMemo(() => config, [config.id, config.type])
useEffect(() => {
  process(stableConfig)
}, [stableConfig])
```

**Détection :** `biome: useExhaustiveDependencies`

---

### 1.4 useMemo / useCallback manquants sur props objet/fonction → AI smell #10

**Pattern interdit :**
```typescript
// ❌ Objet recréé à chaque render → child re-render inutile
function Parent() {
  const options = { timeout: 5000, retry: true } // nouveau objet à chaque render
  return <Child options={options} />
}
```

**Correction :**
```typescript
// ✅ Référence stable
function Parent() {
  const options = useMemo(() => ({ timeout: 5000, retry: true }), [])
  return <Child options={options} />
}
```

---

## Section 2 — Tauri IPC

### 2.1 `file.path || file.name` pour obtenir un chemin → toujours faux en WebView2

**Pattern interdit :**
```typescript
// ❌ Ne fonctionne pas dans WebView2 (Tauri)
const handleDrop = (e: DragEvent) => {
  const path = e.dataTransfer.files[0].path || e.dataTransfer.files[0].name
  // path sera undefined ou le nom seul, jamais le chemin absolu
}
```

**Correction :**
```typescript
// ✅ Utiliser l'API Tauri pour obtenir les chemins
import { open } from '@tauri-apps/api/dialog'
const path = await open({ multiple: false })
// Ou utiliser l'événement tauri://file-drop
```

**Sévérité :** blocking (fonctionne en dev Electron mais échoue silencieusement en WebView2)

---

### 2.2 Réponse IPC non validée par Zod → violation P4

**Pattern interdit :**
```typescript
// ❌ Cast sans validation
const result = await invoke('get_user') as User
// Si le backend change, TypeScript ne détecte rien à runtime
```

**Correction :**
```typescript
// ✅ Validation Zod à la frontière IPC
import { z } from 'zod'
const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email()
})
const raw = await invoke('get_user')
const user = UserSchema.parse(raw) // throw si format inattendu
```

**Détection :** Grep : `await invoke(` suivi de `as [A-Z]` sur la même ligne

---

### 2.3 Pas de resync depuis backend après mutation IPC → violation P2

**Voir Section 1.2** — même pattern, rappel spécifique IPC.

---

## Section 3 — TypeScript

### 3.1 `any` sans commentaire justificatif → violation P4

**Pattern interdit :**
```typescript
// ❌ any sans raison
function processData(data: any) { ... }
const result: any = await fetch(url).then(r => r.json())
```

**Pattern autorisé (avec justification) :**
```typescript
// ✅ any justifié par une contrainte externe
// eslint-disable-next-line @typescript-eslint/no-explicit-any
function adaptLegacyApi(input: any): SafeType {
  // why: la bibliothèque externe v2.x n'exporte pas ses types
  return SafeTypeSchema.parse(input)
}
```

**Détection :** `biome: noExplicitAny` · `tsc --strict`

---

### 3.2 `as SomeType` sur réponse externe → violation P4

**Pattern interdit :**
```typescript
// ❌ Cast mensonger — TypeScript n'a aucune garantie à runtime
const config = JSON.parse(configStr) as AppConfig
const response = await fetch(url).then(r => r.json()) as ApiResponse
```

**Correction :**
```typescript
// ✅ Parse et valide
const config = AppConfigSchema.parse(JSON.parse(configStr))
const response = ApiResponseSchema.parse(await fetch(url).then(r => r.json()))
```

---

### 3.3 tsconfig.json sans `strict: true`

**Pattern interdit :**
```json
{ "compilerOptions": { "target": "ESNext" } }
```

**Correction :**
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

**Sévérité :** important (pas de narrowing automatique, null/undefined non vérifiés)

---

## Section 4 — Hooks React

### 4.1 Hook qui fait plus d'une chose → violation P1

**Pattern interdit :**
```typescript
// ❌ Un seul hook fait fetch + transform + format + persist
function useData(id: string) {
  const [data, setData] = useState(null)
  const [formatted, setFormatted] = useState('')
  const [saved, setSaved] = useState(false)
  useEffect(() => {
    fetch(id).then(raw => {
      const transformed = transform(raw)
      setData(transformed)
      setFormatted(format(transformed))
      persist(transformed).then(() => setSaved(true))
    })
  }, [id])
  return { data, formatted, saved }
}
```

**Correction :**
```typescript
// ✅ Hooks séparés par responsabilité
function useData(id: string) { /* fetch only */ }
function useFormattedData(data: Data) { /* format only */ }
function usePersist(data: Data) { /* persist only */ }
```

---

### 4.2 useEffect sans cleanup → fuite mémoire (AI smell #13)

**Pattern interdit :**
```typescript
useEffect(() => {
  const subscription = store.subscribe(handler) // abonnement créé
  // ❌ Pas de cleanup : la subscription reste active après unmount
}, [])
```

**Correction :**
```typescript
useEffect(() => {
  const subscription = store.subscribe(handler)
  return () => subscription.unsubscribe() // ✅ cleanup obligatoire
}, [])
```

---

### 4.3 State non localisé au plus bas niveau utile → violation P2

**Pattern interdit :**
```typescript
// ❌ State global pour une info locale
const useGlobalStore = create(set => ({
  isModalOpen: false,       // ce state n'intéresse que le composant Modal
  setIsModalOpen: (v) => set({ isModalOpen: v })
}))
```

**Correction :**
```typescript
// ✅ State local dans le composant qui en a besoin
function Modal() {
  const [isOpen, setIsOpen] = useState(false)
  // ...
}
```

---

## Outils de détection (résumé)

| Violation | Outil | Règle |
|-----------|-------|-------|
| Mutation directe | biome | `noDirectMutation` |
| useEffect deps | biome | `useExhaustiveDependencies` |
| any explicite | biome | `noExplicitAny` |
| strict mode | tsc | `--strict` |
| Fichiers couplés | qartez | `qartez_hotspots` |
| Dead code | qartez | `qartez_unused` |
| IPC sans Zod | grep | `invoke(` + `as [A-Z]` |
| file.path WebView2 | grep | `file\.path \|\|` |

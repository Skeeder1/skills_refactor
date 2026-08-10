# Optional playbook — React + TypeScript + Tauri
# Used by: principles-auditor, only when React / TypeScript / Tauri is detected
# Cross-reference: principles.md (P1-P10), ai-smells.md
---

This playbook lists the violations and patterns specific to React + TypeScript + Tauri projects.
It complements `principles.md` with concrete rules tied to that stack.
It must never be applied to a project that does not match this profile.

---

## Section 1 — State management

### 1.1 Direct mutation of React state → P3 violation

**Forbidden pattern:**
```typescript
// ❌ Direct mutation
state.items.push(newItem)
state.items.splice(index, 1)
items[0] = updatedItem
```

**Fix:**
```typescript
// ✅ Immutable return
setItems(prev => [...prev, newItem])
setItems(prev => prev.filter((_, i) => i !== index))
setItems(prev => prev.map((item, i) => i === 0 ? updatedItem : item))
```

**Detection:** `biome: noDirectMutation` · Grep: `\.push\(|\.splice\(|\.shift\(` inside React handlers

---

### 1.2 State duplicated between Zustand and the Tauri backend → P2 violation

**Forbidden pattern:**
```typescript
// ❌ Optimistic update with no reconciliation
const addEntry = async (entry: Entry) => {
  setEntries([...entries, entry])  // optimistic local update
  await invoke('add_entry', { entry }) // no resync from the response
}
```

**Fix:**
```typescript
// ✅ Single source of truth, from the backend
const addEntry = async (entry: Entry) => {
  const updatedEntries = await invoke<Entry[]>('add_entry', { entry })
  setEntries(updatedEntries)  // the backend response is authoritative
}
```

**Detection:** Grep: `invoke(` + `set[A-Z]` without reading the result of the invoke in the same function

---

### 1.3 useEffect with incorrect deps → P3 violation

**Forbidden pattern:**
```typescript
// ❌ Empty deps while the effect depends on props/state
useEffect(() => {
  fetchData(userId) // userId changes but the effect never re-runs
}, [])

// ❌ Deps too broad, causing an infinite loop
useEffect(() => {
  process(config)
}, [config]) // if config is an object recreated on every render
```

**Fix:**
```typescript
// ✅ Precise deps
useEffect(() => {
  fetchData(userId)
}, [userId])

// ✅ Stabilise with useMemo when config is an object
const stableConfig = useMemo(() => config, [config.id, config.type])
useEffect(() => {
  process(stableConfig)
}, [stableConfig])
```

**Detection:** `biome: useExhaustiveDependencies`

---

### 1.4 Missing useMemo / useCallback on object or function props → AI smell #10

**Forbidden pattern:**
```typescript
// ❌ Object recreated on every render → pointless child re-render
function Parent() {
  const options = { timeout: 5000, retry: true } // a new object on every render
  return <Child options={options} />
}
```

**Fix:**
```typescript
// ✅ Stable reference
function Parent() {
  const options = useMemo(() => ({ timeout: 5000, retry: true }), [])
  return <Child options={options} />
}
```

---

## Section 2 — Tauri IPC

### 2.1 `file.path || file.name` to obtain a path → always wrong in WebView2

**Forbidden pattern:**
```typescript
// ❌ Does not work inside WebView2 (Tauri)
const handleDrop = (e: DragEvent) => {
  const path = e.dataTransfer.files[0].path || e.dataTransfer.files[0].name
  // path will be undefined, or the bare name — never the absolute path
}
```

**Fix:**
```typescript
// ✅ Use the Tauri API to obtain paths
import { open } from '@tauri-apps/api/dialog'
const path = await open({ multiple: false })
// Or use the tauri://file-drop event
```

**Severity:** blocking (works in Electron dev but fails silently in WebView2)

---

### 2.2 IPC response not validated with Zod → P4 violation

**Forbidden pattern:**
```typescript
// ❌ Cast without validation
const result = await invoke('get_user') as User
// If the backend changes, TypeScript detects nothing at runtime
```

**Fix:**
```typescript
// ✅ Zod validation at the IPC boundary
import { z } from 'zod'
const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email()
})
const raw = await invoke('get_user')
const user = UserSchema.parse(raw) // throws on an unexpected shape
```

**Detection:** Grep: `await invoke(` followed by `as [A-Z]` on the same line

---

### 2.3 No resync from the backend after an IPC mutation → P2 violation

**See Section 1.2** — same pattern, restated for IPC.

---

## Section 3 — TypeScript

### 3.1 `any` with no justifying comment → P4 violation

**Forbidden pattern:**
```typescript
// ❌ any with no reason
function processData(data: any) { ... }
const result: any = await fetch(url).then(r => r.json())
```

**Allowed pattern (with justification):**
```typescript
// ✅ any justified by an external constraint
// eslint-disable-next-line @typescript-eslint/no-explicit-any
function adaptLegacyApi(input: any): SafeType {
  // why: the external library v2.x does not export its types
  return SafeTypeSchema.parse(input)
}
```

**Detection:** `biome: noExplicitAny` · `tsc --strict`

---

### 3.2 `as SomeType` on an external response → P4 violation

**Forbidden pattern:**
```typescript
// ❌ A lying cast — TypeScript guarantees nothing at runtime
const config = JSON.parse(configStr) as AppConfig
const response = await fetch(url).then(r => r.json()) as ApiResponse
```

**Fix:**
```typescript
// ✅ Parse and validate
const config = AppConfigSchema.parse(JSON.parse(configStr))
const response = ApiResponseSchema.parse(await fetch(url).then(r => r.json()))
```

---

### 3.3 tsconfig.json without `strict: true`

**Forbidden pattern:**
```json
{ "compilerOptions": { "target": "ESNext" } }
```

**Fix:**
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

**Severity:** important (no automatic narrowing, null/undefined unchecked)

---

## Section 4 — React hooks

### 4.1 A hook doing more than one thing → P1 violation

**Forbidden pattern:**
```typescript
// ❌ A single hook does fetch + transform + format + persist
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

**Fix:**
```typescript
// ✅ Hooks split by responsibility
function useData(id: string) { /* fetch only */ }
function useFormattedData(data: Data) { /* format only */ }
function usePersist(data: Data) { /* persist only */ }
```

---

### 4.2 useEffect without cleanup → memory leak (AI smell #13)

**Forbidden pattern:**
```typescript
useEffect(() => {
  const subscription = store.subscribe(handler) // subscription created
  // ❌ No cleanup: the subscription stays active after unmount
}, [])
```

**Fix:**
```typescript
useEffect(() => {
  const subscription = store.subscribe(handler)
  return () => subscription.unsubscribe() // ✅ cleanup is mandatory
}, [])
```

---

### 4.3 State not kept at the lowest useful level → P2 violation

**Forbidden pattern:**
```typescript
// ❌ Global state for local information
const useGlobalStore = create(set => ({
  isModalOpen: false,       // this state only concerns the Modal component
  setIsModalOpen: (v) => set({ isModalOpen: v })
}))
```

**Fix:**
```typescript
// ✅ Local state, in the component that needs it
function Modal() {
  const [isOpen, setIsOpen] = useState(false)
  // ...
}
```

---

## Detection tooling (summary)

| Violation | Tool | Rule |
|-----------|------|------|
| Direct mutation | biome | `noDirectMutation` |
| useEffect deps | biome | `useExhaustiveDependencies` |
| Explicit any | biome | `noExplicitAny` |
| strict mode | tsc | `--strict` |
| Coupled files | qartez | `qartez_hotspots` |
| Dead code | qartez | `qartez_unused` |
| IPC without Zod | grep | `invoke(` + `as [A-Z]` |
| file.path in WebView2 | grep | `file\.path \|\|` |

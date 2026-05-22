# Safe refactor rules
# Utilisé par : refactor-executor
# Ce fichier définit ce qui peut être modifié automatiquement et ce qui est interdit.
---

## Catégorie 1 — Toujours sûr (sans confirmation)

Ces opérations peuvent être appliquées directement sans demander de validation humaine :

- **Supprimer du code mort confirmé** : symbole avec 0 importeurs vérifié cross-fichiers
  (confirmation par `qartez_unused` ou `knip`, pas seulement par analyse locale)
- **Supprimer les logs de debug** : `console.log`, `console.error`, `console.warn`,
  `dbg!()`, `print!()` en dehors des fichiers de test
- **Supprimer les blocs commentés** : blocs de code commenté > 3 lignes consécutives
- **Extraire une constante nommée** : magic number ou magic string dans le même fichier,
  sans changer la logique ou la signature
- **Ajouter ou corriger un JSDoc/docstring** : sans modifier la signature de la fonction

---

## Catégorie 2 — Sûr avec validation post-modification

Ces opérations doivent passer la validation (`tsc --noEmit && biome check` ou `cargo clippy`)
avant d'être considérées comme appliquées. Si la validation échoue → revert immédiat.

- **Renommer un symbole exporté** + mettre à jour tous les importeurs dans le même passage
  (utiliser `qartez_refs` pour lister tous les usages, puis `MultiEdit` pour le rename)
- **Extraire une fonction** dans le même fichier (sans changer le comportement observable)
- **Remplacer `array.push(x)`** par `setArray([...array, x])` dans un contexte React
- **Ajouter des types TypeScript manquants** sur des paramètres ou retours de fonctions
  (doit passer `tsc --noEmit`)
- **Déplacer une fonction** vers un fichier `utils` + mettre à jour tous les imports
- **Remplacer un `any`** par un type précis + narrowing (doit passer `tsc --noEmit`)
- **Ajouter une fonction de cleanup** à un `useEffect` qui crée un abonnement ou timer

---

## Catégorie 3 — Jamais sans confirmation explicite de l'utilisateur

Ces opérations ont un impact trop large ou trop risqué pour être automatisées :

- Changer la signature d'une fonction publique (paramètres, ordre, types)
- Modifier le type de retour d'une fonction
- Supprimer un fichier entier (même si apparemment inutilisé)
- Modifier des fichiers d'authentification, sécurité, tokens, sessions
- Toucher les migrations ou le schéma de base de données
- Modifier les fichiers de tests au-delà de la mise à jour des imports
- Changer la configuration de build (`vite.config`, `tsconfig`, `Cargo.toml` sections critiques)
- Toucher des fichiers avec `// DO NOT EDIT` ou `// GENERATED`

---

## Catégorie 4 — Jamais toucher (liste noire absolue)

Ces fichiers ne doivent JAMAIS être modifiés par refactor-executor, quelle que soit la violation :

- `dist/`, `build/`, `.next/`, `out/` (fichiers générés)
- `*.generated.*`, `*.gen.ts`, `*.gen.rs`
- `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Cargo.lock`
- Tout fichier marqué `// DO NOT EDIT` ou `/* DO NOT EDIT */`
- Tout fichier listé dans `violations.manual_verify`
- Fichiers de migrations (`migrations/`, `*.migration.ts`, `*.up.sql`)

---

## Protocole par fichier (5 étapes obligatoires)

Pour chaque fichier à modifier dans `violations.blocking` + `violations.important` :

**Étape 1 — Vérification du safety gate**
```
SI fichier dans violations.manual_verify → SKIP, log dans changes.skipped avec raison "manual-verify"
SI fichier dans la liste noire absolue → SKIP, log dans changes.skipped avec raison "blacklisted"
```

**Étape 2 — Vérification du blast radius (si Qartez disponible)**
```
SI fichier dans findings.hotspots avec score > 5 :
  → appeler qartez_impact(fichier)
  → SI impact.transitive_dependents > 20 ET changement non trivial (pas dead code) :
    → SKIP, log dans changes.skipped avec raison "blast-radius-too-high"
  → Sinon : noter le blast_radius dans changes.applied
```

**Étape 3 — Cross-fichiers (si rename)**
```
SI le changement est un rename cross-fichiers ET Qartez disponible :
  → appeler qartez_refs(symbole) pour lister tous les usages
  → renommer dans tous les fichiers en un seul passage avec MultiEdit
```

**Étape 4 — Appliquer le changement**
```
Appliquer le changement le plus petit possible.
Ne pas profiter de l'ouverture du fichier pour nettoyer d'autres zones non listées.
```

**Étape 5 — Validation**
```
TypeScript/JS :
  → npx tsc --noEmit
  → npx biome check <fichier>

Rust :
  → cargo clippy

SI validation OK :
  → log dans changes.applied avec validated: true, tools_passed: [...]

SI validation KO :
  → inspecter git diff pour comprendre l'échec
  → revert le fichier à son état d'origine
  → log dans changes.skipped avec raison "reverted:tsc-failed|biome-failed|clippy-failed"
    et inclure le message d'erreur dans validation_output

Passer au fichier suivant.
```

---

## Règle de progression

Ne jamais bloquer la chaîne sur un seul fichier. Si un fichier échoue :
1. Log l'échec dans `changes.skipped` avec raison détaillée
2. Continuer avec le fichier suivant
3. Laisser le `doc-updater` et le rapport final noter les fichiers skipped

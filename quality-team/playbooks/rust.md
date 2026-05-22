# Playbook optionnel — Rust + Tauri backend
# Utilisé par : principles-auditor uniquement si Rust est détecté
# Référence croisée : principles.md (P1-P10), ai-smells.md
---

Ce playbook liste les violations et patterns spécifiques aux projets Rust + Tauri.
Il complète `principles.md` avec des règles concrètes pour le code Rust côté backend.
Il ne doit jamais être appliqué à un projet sans Rust détecté.

---

## Section 1 — Gestion d'erreurs

### 1.1 `unwrap()` / `expect()` sur des chemins utilisateur → violation P5

**Pattern interdit :**
```rust
// ❌ Panique si le fichier n'existe pas — crash de l'application
let content = std::fs::read_to_string(path).unwrap()

// ❌ expect() = unwrap() avec un message, mais panique quand même
let config = serde_json::from_str::<Config>(&json).expect("config invalide")
```

**Correction :**
```rust
// ✅ Propagation avec ? ou gestion explicite
let content = std::fs::read_to_string(path)
    .map_err(|e| AppError::FileRead { path: path.to_string(), source: e })?;

// ✅ Ou matching explicite
let config = match serde_json::from_str::<Config>(&json) {
    Ok(c) => c,
    Err(e) => return Err(AppError::ConfigParse(e.to_string())),
};
```

**Règle Clippy :** `clippy::unwrap_used`, `clippy::expect_used`

---

### 1.2 `let _ = result_qui_peut_echouer` → violation P5

**Pattern interdit :**
```rust
// ❌ L'erreur est silencieusement ignorée
let _ = write_log(message);
let _ = db.save(&record);
```

**Correction :**
```rust
// ✅ Propagation ou gestion explicite
write_log(message)?;

// ✅ Ou logging de l'erreur si on ne peut pas propager
if let Err(e) = db.save(&record) {
    tracing::error!("Échec save: {e}");
}
```

---

### 1.3 Commande Tauri sans `Result<T, E>` → violation P5

**Pattern interdit :**
```rust
// ❌ Aucun moyen de signaler une erreur au frontend
#[tauri::command]
fn get_user(id: String) -> User {
    db::find_user(&id).unwrap() // panique si absent
}
```

**Correction :**
```rust
// ✅ Toujours retourner Result depuis une commande Tauri
#[tauri::command]
fn get_user(id: String) -> Result<User, String> {
    db::find_user(&id).map_err(|e| e.to_string())
}

// ✅ Encore mieux : un type d'erreur dédié
#[tauri::command]
fn get_user(id: String) -> Result<User, AppError> {
    db::find_user(&id).map_err(AppError::from)
}
```

**Règle :** Toute fonction annotée `#[tauri::command]` doit avoir `Result<T, E>` comme type de retour.

---

### 1.4 `catch { continue }` implicite — patterns Rust équivalents

```rust
// ❌ Ignorer silencieusement via if let (acceptable seulement si vraiment optionnel)
if let Ok(result) = risky_operation() {
    use_result(result)
}
// Si risky_operation() échoue sur des données utilisateur → silent failure

// ✅ Toujours logger au minimum
match risky_operation() {
    Ok(result) => use_result(result),
    Err(e) => tracing::warn!("Opération échouée (non critique): {e}"),
}
```

---

## Section 2 — Nommage et structure

### 2.1 Convention snake_case obligatoire

```rust
// ❌ camelCase (Rust warning)
fn getUserById(id: &str) -> User { ... }
struct UserData { firstName: String }

// ✅ snake_case partout
fn get_user_by_id(id: &str) -> User { ... }
struct UserData { first_name: String }
```

**Détection :** `cargo clippy` (non_snake_case warning by default)

---

### 2.2 `pub use` dans `mod.rs` pour une API propre → violation P1 si absent

```rust
// Structure recommandée :
// src/
//   users/
//     mod.rs      ← API publique du module uniquement
//     repository.rs
//     service.rs
//     types.rs

// ✅ mod.rs expose uniquement l'interface publique
pub use self::service::UserService;
pub use self::types::{User, CreateUserRequest, UserError};
// ❌ Ne pas mettre de logique dans mod.rs
```

---

### 2.3 Logique dans `mod.rs` → violation P1

```rust
// ❌ mod.rs avec de la logique métier
pub mod users;

pub fn process_user(user: &User) -> Result<(), Error> {
    // 50 lignes de logique ici
}
```

**Correction :** Déplacer toute logique dans des fichiers dédiés (`service.rs`, `handler.rs`, etc.)

---

## Section 3 — Clippy obligatoire

### 3.1 Configuration Clippy recommandée

Dans `.cargo/config.toml` ou `Cargo.toml` :

```toml
[lints.clippy]
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
indexing_slicing = "warn"
```

### 3.2 Lints critiques à activer

| Lint | Niveau | Raison |
|------|--------|--------|
| `clippy::unwrap_used` | deny | Panique sur chemin utilisateur |
| `clippy::expect_used` | deny | Idem |
| `clippy::panic` | deny | Jamais de panic en production Tauri |
| `clippy::indexing_slicing` | warn | `slice[n]` peut paniquer |
| `clippy::todo` | warn | Code incomplet en production |
| `clippy::unimplemented` | warn | Idem |
| `clippy::unwrap_in_result` | deny | unwrap à l'intérieur d'un Result |

### 3.3 CI obligatoire

```bash
# ❌ Ne pas utiliser sans -D warnings en CI
cargo clippy

# ✅ CI strict — échoue si warning
cargo clippy -- -D warnings
```

---

## Section 4 — Paths et fichiers

### 4.1 `PathBuf::from("nom.pdf").exists()` sans chemin absolu → toujours faux

**Pattern interdit :**
```rust
// ❌ Chemin relatif non résolu — exists() retournera false en production
let path = PathBuf::from("document.pdf");
if path.exists() { ... }

// ❌ Chemin construit depuis une string sans validation
fn process_file(filename: &str) -> Result<()> {
    let path = PathBuf::from(filename)
    // filename peut être "../../../etc/passwd"
}
```

**Correction :**
```rust
// ✅ Résoudre depuis le répertoire de données Tauri
use tauri::api::path::app_data_dir;
let base = app_data_dir(&config).ok_or(AppError::PathResolution)?;
let path = base.join("document.pdf");
if path.exists() { ... }

// ✅ Valider et canonicaliser
fn process_file(base_dir: &Path, filename: &str) -> Result<PathBuf> {
    let path = base_dir.join(filename);
    let canonical = path.canonicalize()
        .map_err(|_| AppError::InvalidPath)?;
    // Vérifier que le chemin est bien sous base_dir (path traversal protection)
    if !canonical.starts_with(base_dir) {
        return Err(AppError::PathTraversal);
    }
    Ok(canonical)
}
```

**Sévérité :** blocking (bug silencieux + risque sécurité)

---

### 4.2 Chemins construits depuis une string sans validation → violation P4

```rust
// ❌ Concaténation directe sans validation
fn open_user_file(user_input: &str) -> Result<File> {
    let path = format!("/data/{}", user_input) // path traversal possible
    File::open(path).map_err(AppError::from)
}

// ✅ Validation stricte
fn open_user_file(base: &Path, filename: &str) -> Result<File> {
    // Rejeter les chemins contenant des séparateurs
    if filename.contains('/') || filename.contains('\\') || filename.contains("..") {
        return Err(AppError::InvalidFilename);
    }
    let path = base.join(filename);
    File::open(path).map_err(AppError::from)
}
```

---

## Section 5 — Type d'erreur domaine

### 5.1 Définir un type d'erreur applicatif centralisé

```rust
// ✅ Un enum d'erreur par domaine ou par module
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("Fichier introuvable : {path}")]
    FileNotFound { path: String },

    #[error("Erreur de sérialisation : {0}")]
    Serialization(#[from] serde_json::Error),

    #[error("Erreur IPC : {0}")]
    Ipc(String),

    #[error("Accès refusé")]
    Unauthorized,
}

// ✅ Implémentation de serde::Serialize pour Tauri
impl serde::Serialize for AppError {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: serde::Serializer {
        serializer.serialize_str(self.to_string().as_ref())
    }
}
```

---

## Outils de détection (résumé)

| Violation | Outil | Règle / Grep |
|-----------|-------|-------------|
| unwrap/expect | clippy | `clippy::unwrap_used`, `clippy::expect_used` |
| Commande sans Result | grep | `#\[tauri::command\]` sans `Result<` |
| Panique | clippy | `clippy::panic` |
| Chemin relatif | grep | `PathBuf::from("` + pas de variable |
| let _ ignoré | clippy | `clippy::let_underscore_must_use` |
| non snake_case | clippy | warning par défaut |
| Logique dans mod.rs | qartez | `qartez_outline` sur mod.rs |

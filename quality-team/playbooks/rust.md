# Optional playbook — Rust + Tauri backend
# Used by: principles-auditor, only when Rust is detected
# Cross-reference: principles.md (P1-P10), ai-smells.md
---

This playbook lists the violations and patterns specific to Rust + Tauri projects.
It complements `principles.md` with concrete rules for backend-side Rust code.
It must never be applied to a project where Rust was not detected.

---

## Section 1 — Error handling

### 1.1 `unwrap()` / `expect()` on user-facing paths → P5 violation

**Forbidden pattern:**
```rust
// ❌ Panics if the file does not exist — the application crashes
let content = std::fs::read_to_string(path).unwrap()

// ❌ expect() = unwrap() with a message, but it still panics
let config = serde_json::from_str::<Config>(&json).expect("invalid config")
```

**Fix:**
```rust
// ✅ Propagate with ?, or handle explicitly
let content = std::fs::read_to_string(path)
    .map_err(|e| AppError::FileRead { path: path.to_string(), source: e })?;

// ✅ Or match explicitly
let config = match serde_json::from_str::<Config>(&json) {
    Ok(c) => c,
    Err(e) => return Err(AppError::ConfigParse(e.to_string())),
};
```

**Clippy rule:** `clippy::unwrap_used`, `clippy::expect_used`

---

### 1.2 `let _ = fallible_result` → P5 violation

**Forbidden pattern:**
```rust
// ❌ The error is silently discarded
let _ = write_log(message);
let _ = db.save(&record);
```

**Fix:**
```rust
// ✅ Propagate, or handle explicitly
write_log(message)?;

// ✅ Or log the error when propagation is not possible
if let Err(e) = db.save(&record) {
    tracing::error!("Save failed: {e}");
}
```

---

### 1.3 A Tauri command without `Result<T, E>` → P5 violation

**Forbidden pattern:**
```rust
// ❌ No way to report an error to the frontend
#[tauri::command]
fn get_user(id: String) -> User {
    db::find_user(&id).unwrap() // panics when missing
}
```

**Fix:**
```rust
// ✅ Always return a Result from a Tauri command
#[tauri::command]
fn get_user(id: String) -> Result<User, String> {
    db::find_user(&id).map_err(|e| e.to_string())
}

// ✅ Better still: a dedicated error type
#[tauri::command]
fn get_user(id: String) -> Result<User, AppError> {
    db::find_user(&id).map_err(AppError::from)
}
```

**Rule:** every function annotated `#[tauri::command]` must have `Result<T, E>` as its return type.

---

### 1.4 The implicit `catch { continue }` — equivalent Rust patterns

```rust
// ❌ Silently ignoring via if let (acceptable only when it really is optional)
if let Ok(result) = risky_operation() {
    use_result(result)
}
// If risky_operation() fails on user data → silent failure

// ✅ Always log, at minimum
match risky_operation() {
    Ok(result) => use_result(result),
    Err(e) => tracing::warn!("Operation failed (non-critical): {e}"),
}
```

---

## Section 2 — Naming and structure

### 2.1 snake_case convention is mandatory

```rust
// ❌ camelCase (Rust warning)
fn getUserById(id: &str) -> User { ... }
struct UserData { firstName: String }

// ✅ snake_case everywhere
fn get_user_by_id(id: &str) -> User { ... }
struct UserData { first_name: String }
```

**Detection:** `cargo clippy` (non_snake_case warning by default)

---

### 2.2 `pub use` in `mod.rs` for a clean API → P1 violation when missing

```rust
// Recommended structure:
// src/
//   users/
//     mod.rs      ← the module's public API only
//     repository.rs
//     service.rs
//     types.rs

// ✅ mod.rs exposes the public interface only
pub use self::service::UserService;
pub use self::types::{User, CreateUserRequest, UserError};
// ❌ Do not put logic in mod.rs
```

---

### 2.3 Logic inside `mod.rs` → P1 violation

```rust
// ❌ mod.rs holding business logic
pub mod users;

pub fn process_user(user: &User) -> Result<(), Error> {
    // 50 lines of logic here
}
```

**Fix:** move all logic into dedicated files (`service.rs`, `handler.rs`, and so on).

---

## Section 3 — Clippy is mandatory

### 3.1 Recommended Clippy configuration

In `.cargo/config.toml` or `Cargo.toml`:

```toml
[lints.clippy]
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
indexing_slicing = "warn"
```

### 3.2 Critical lints to enable

| Lint | Level | Reason |
|------|-------|--------|
| `clippy::unwrap_used` | deny | Panics on a user-facing path |
| `clippy::expect_used` | deny | Same |
| `clippy::panic` | deny | Never panic in a Tauri production build |
| `clippy::indexing_slicing` | warn | `slice[n]` can panic |
| `clippy::todo` | warn | Incomplete code in production |
| `clippy::unimplemented` | warn | Same |
| `clippy::unwrap_in_result` | deny | unwrap inside a Result |

### 3.3 CI is mandatory

```bash
# ❌ Do not use it without -D warnings in CI
cargo clippy

# ✅ Strict CI — fails on a warning
cargo clippy -- -D warnings
```

---

## Section 4 — Paths and files

### 4.1 `PathBuf::from("name.pdf").exists()` without an absolute path → always false

**Forbidden pattern:**
```rust
// ❌ Unresolved relative path — exists() will return false in production
let path = PathBuf::from("document.pdf");
if path.exists() { ... }

// ❌ Path built from a string with no validation
fn process_file(filename: &str) -> Result<()> {
    let path = PathBuf::from(filename)
    // filename could be "../../../etc/passwd"
}
```

**Fix:**
```rust
// ✅ Resolve from the Tauri data directory
use tauri::api::path::app_data_dir;
let base = app_data_dir(&config).ok_or(AppError::PathResolution)?;
let path = base.join("document.pdf");
if path.exists() { ... }

// ✅ Validate and canonicalise
fn process_file(base_dir: &Path, filename: &str) -> Result<PathBuf> {
    let path = base_dir.join(filename);
    let canonical = path.canonicalize()
        .map_err(|_| AppError::InvalidPath)?;
    // Check the path really is under base_dir (path traversal protection)
    if !canonical.starts_with(base_dir) {
        return Err(AppError::PathTraversal);
    }
    Ok(canonical)
}
```

**Severity:** blocking (silent bug + security risk)

---

### 4.2 Paths built from a string without validation → P4 violation

```rust
// ❌ Direct concatenation with no validation
fn open_user_file(user_input: &str) -> Result<File> {
    let path = format!("/data/{}", user_input) // path traversal possible
    File::open(path).map_err(AppError::from)
}

// ✅ Strict validation
fn open_user_file(base: &Path, filename: &str) -> Result<File> {
    // Reject paths containing separators
    if filename.contains('/') || filename.contains('\\') || filename.contains("..") {
        return Err(AppError::InvalidFilename);
    }
    let path = base.join(filename);
    File::open(path).map_err(AppError::from)
}
```

---

## Section 5 — Domain error type

### 5.1 Define one centralised application error type

```rust
// ✅ One error enum per domain or per module
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("File not found: {path}")]
    FileNotFound { path: String },

    #[error("Serialisation error: {0}")]
    Serialization(#[from] serde_json::Error),

    #[error("IPC error: {0}")]
    Ipc(String),

    #[error("Access denied")]
    Unauthorized,
}

// ✅ serde::Serialize implementation, for Tauri
impl serde::Serialize for AppError {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: serde::Serializer {
        serializer.serialize_str(self.to_string().as_ref())
    }
}
```

---

## Detection tooling (summary)

| Violation | Tool | Rule / Grep |
|-----------|------|-------------|
| unwrap/expect | clippy | `clippy::unwrap_used`, `clippy::expect_used` |
| Command without Result | grep | `#\[tauri::command\]` without `Result<` |
| Panic | clippy | `clippy::panic` |
| Relative path | grep | `PathBuf::from("` with no variable |
| Ignored let _ | clippy | `clippy::let_underscore_must_use` |
| Non snake_case | clippy | warning by default |
| Logic in mod.rs | qartez | `qartez_outline` on mod.rs |

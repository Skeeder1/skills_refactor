# Core principles — quality-team
# Used by: principles-auditor
# Universal reference, independent of language and framework.
---

These principles apply to any codebase. Anything specific to a language,
framework or runtime belongs in an optional playbook, never in this file.

## P1 — Single responsibility

**Rule:** A file, module, type or function must have one main reason to change.

**Detectable violations:**
- A module mixing orchestration, data access, rendering, validation and I/O.
- A function doing setup, validation, computation, persistence and reporting.
- A very long file covering several unrelated business or technical domains.

**Standard fix:**
- Extract the responsibilities into named modules or functions.
- Separate orchestration, business rules, external access and presentation.
- Keep public interfaces small and intentional.

**Automatic detection:**
- Qartez hotspots, Lizard complexity, dependency graph, number of exports.

## P2 — Single source of truth

**Rule:** A piece of business or technical information must have one
authoritative source. Copies, caches and derived views must be explicitly
synchronised or recomputable.

**Detectable violations:**
- The same value maintained in two structures with no clear authoritative source.
- A cache updated by hand, with no invalidation.
- Derived state stored instead of computed.
- Configuration duplicated across code, file and environment.

**Standard fix:**
- Pick the authoritative source.
- Derive the secondary views.
- Centralise the configuration and document the invalidation rules.

**Automatic detection:**
- Qartez/Lizard clones, duplicated constants, frequent co-change.

## P3 — Explicit contracts at the boundaries

**Rule:** Any data coming from an external boundary must be validated,
normalised or typed according to the conventions of the language.

**Boundaries:**
- API, files, database, CLI, environment variables, network, inter-process
  messages, serialisation, user input.

**Detectable violations:**
- External data used directly as trusted internal data.
- A cast/assertion with no runtime validation where the language requires one.
- Parsing with no error handling.
- An implicit, undocumented format.

**Standard fix:**
- Introduce a parser, a validator, a DTO or a domain type.
- Reject or normalise invalid data at the boundary.
- Document the public contract.

**Automatic detection:**
- Linters, typecheckers, grep on parse/cast, security rules, contract tests.

## P4 — Mutation integrity and invariants

**Rule:** Every mutation must preserve the domain invariants and stay local.
Shared or observable state must be modified in a controlled way.

**Detectable violations:**
- Direct mutation of a shared object, bypassing the owning API.
- A partial update that leaves an invariant broken when an error occurs.
- Data modified in place while consumers expect a new value.

**Standard fix:**
- Centralise mutations in a domain function or method.
- Use transactions, controlled copies or guards, depending on the language.
- Validate the invariants before and after a critical mutation.

**Automatic detection:**
- Linters, AST analysis, invariant tests, hotspot review.

## P5 — Explicit errors

**Rule:** An error must be handled, propagated or converted into a domain error.
It must never disappear silently.

**Detectable violations:**
- An empty catch, or one returning a neutral value with no justification.
- The result of a critical operation ignored.
- A possible exception/panic/abort on user input.
- A generic error message that loses the useful cause.

**Standard fix:**
- Propagate the error, or map it to a domain type/format.
- Log only when the log is actionable and does not replace propagation.
- Add an explicit fallback strategy for the errors you accept.

**Automatic detection:**
- Linters, analysis of catch/match/result blocks, failure tests.

## P6 — Isolated side effects

**Rule:** Side effects must be explicit, local, and separated from pure
computation whenever that is reasonable.

**Detectable violations:**
- A computation function that writes a file, mutates a global or hits the network.
- I/O hidden inside a getter, a formatter or a validator.
- An implicit execution order required for correctness.

**Standard fix:**
- Separate pure computation from orchestration.
- Inject the external dependencies.
- Name effectful functions with explicit verbs.

**Automatic detection:**
- I/O calls inside transformation functions, call graphs, naming review.

## P7 — Duplication under control

**Rule:** Two similar occurrences are a signal; three occurrences usually
justify an extraction. The abstraction must not create more coupling than the
duplication it removes.

**Detectable violations:**
- The same validation → transformation → save sequence in several files.
- Copied business constants or messages.
- Tests or handlers duplicating the same logic.

**Standard fix:**
- Extract a shared function, module or type when the concept really is shared.
- Keep them separate when the domains diverge.

**Automatic detection:**
- `qartez_clones`, Lizard duplicate, constant search.

## P8 — Intentional naming

**Rule:** Names must expose the business or technical intent, not just the shape
of the data.

**Detectable violations:**
- Vague names in non-trivial code: `data`, `value`, `result`, `tmp`, `manager`.
- The same concept named differently across modules.
- A function named like an event while it transforms or persists.

**Standard fix:**
- Rename using the vocabulary of the domain.
- Use one term per concept.
- Prefer precise verbs for actions.

**Automatic detection:**
- Search for generic names, consistency review, Qartez refs before a rename.

## P9 — Small size and low complexity

**Rule:** Units of code must stay readable, testable and shallowly nested.

**Default thresholds:**
- Function > 40 lines: inspect.
- Cyclomatic complexity > 10: inspect.
- Parameters > 4: consider a parameter object/struct.
- Nesting > 3 levels: prefer guards or extraction.

**Standard fix:**
- Extract the phases.
- Replace nesting with guard clauses.
- Introduce a parameter object when it clarifies the call site.

**Automatic detection:**
- Lizard, Qartez hotspots, complexity linters.

## P10 — Living documentation

**Rule:** Documentation must explain the contracts, decisions and constraints
that are not obvious from the code.

**Detectable violations:**
- A public API with no understandable contract.
- A README or AGENTS file referencing stale paths.
- A comment describing the code instead of explaining the why.
- A TODO with no context, owner or resolution condition.

**Standard fix:**
- Document public APIs in the idiomatic format of the language.
- Update the documentation when a module moves.
- Delete pointless narrative comments.

**Automatic detection:**
- Search for undocumented public exports, broken links, TODOs without context.

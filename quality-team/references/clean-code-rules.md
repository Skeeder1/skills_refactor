# Clean code rules — source : ciembor/agent-rules-books
# Utilisé par : principles-auditor
# Licence : MIT
---

# OBEY Clean Code by Robert C. Martin

## When to use

Use when readability, local reasoning, and maintainable code shape are the main concerns, especially during everyday implementation and review.

## Primary bias to correct

Working code is not automatically clean code.

## Decision rules

- Treat cleanliness as part of delivery. Preserve behavior, leave touched code cleaner within scope, and do not add mess because the schedule is tight or a rewrite is promised.
- Write for local reasoning. A reader should understand the path without reconstructing hidden state, wide jumps, or naming trivia.
- Use precise names and one term per concept. Rename code when vocabulary hides intent, overloads meaning, or forces comments to compensate.
- Keep functions small, focused, and at one level of abstraction. Tell the story top-down so intent appears before detail.
- Keep parameters few and meaningful. Avoid boolean flags, output parameters, and grab-bag argument lists; model the concept instead.
- Separate commands from queries and eliminate hidden side effects. A function that answers should not also mutate behind the reader's back.
- Keep the happy path readable. Isolate error handling, invalid-state handling, and cleanup; prefer explicit optionality or typed results over null-like sentinel flow when the language supports it.
- Expose behavior rather than raw representation. Avoid train-wreck access, utility dumping grounds, and classes or modules with mixed responsibilities.
- Keep construction, framework, persistence, transaction, security, and vendor details outside business behavior.
- Make public APIs small, explicit, and hard to misuse. Encode boundary logic, required order, and likely changes where readers can see them.
- Use comments only for rationale, constraints, warnings, or external contracts. Do not narrate code instead of improving it.
- Treat tests as production code: readable, deterministic, aligned with the behavior or contract they protect, and backed by proportionate validation before calling the change done.
- Let design emerge through tests, duplication removal, expressiveness, and minimal structure; do not add needless abstractions or infrastructure.
- When touching code, remove the smell that most increases change cost, but do not silently broaden the task beyond the smallest cleanup that makes the requested change safe.

## Trigger rules

- When a function mixes setup, validation, computation, and side effects, split the phases.
- When a comment explains control flow, simplify names or structure before keeping the comment.
- When a function both mutates and answers, or hides a mode switch behind a flag, separate the responsibilities.
- When duplication, repeated switches, or primitive clusters appear, name the concept with an argument object, polymorphism, special case, or other small abstraction.
- When a boundary leaks framework, vendor, or persistence quirks inward, add or strengthen a local adapter.
- When async or concurrency enters, isolate threading policy, minimize shared mutable state, define shutdown, and test timing-sensitive behavior.
- When fixing a bug or changing behavior, add or update the test that protects the intended contract.
- When cleanup starts spreading into unrelated areas, cut back to the smallest refactor that keeps the requested change safe and readable.

## Final checklist

- Can a reader follow the change locally?
- Are names and APIs carrying the meaning without narration?
- Is mutation explicit and the happy path still clear?
- Did framework, persistence, vendor, and construction details stay behind boundaries?
- Did I remove at least one smell from the touched area?
- Do tests protect the changed behavior or contract?
- Did I actually run the relevant tests or checks for this change?

---

# Supplément — Clean Architecture (source : ciembor/agent-rules-books)

# OBEY Clean Architecture by Robert C. Martin

## When to use

Use when adding, changing, reviewing, or refactoring code whose business rules should survive changes in frameworks, databases, delivery mechanisms, services, devices, vendors, deployment shape, or schedule pressure.

## Primary bias to correct

Do not let details become the architecture. Business policy stays independent, dependencies point inward, and volatile mechanisms remain replaceable.

## Decision rules

- Preserve independent business rules, inward dependencies, testability, and replaceable details even when the immediate feature would be shorter without them.
- Source dependencies must point inward toward higher-level policy. Domain and use cases must not import frameworks, databases, web handlers, queues, external service clients, UI types, or other details.
- Put enterprise rules and invariants in entities or equivalent domain objects; put application-specific orchestration in focused use cases.
- Pass plain request and response models across use-case boundaries. Do not pass web requests, framework contexts, ORM rows, database-bound structures, or framework response objects into or out of core policy.
- Treat frameworks, databases, web delivery, messaging, filesystems, clocks, service clients, networks, devices, and vendors as outer-layer details behind ports, gateways, presenters, mappers, or adapters.
- Inner layers own the interfaces they need; outer layers implement them. Object construction and concrete wiring belong in the composition root or other outer-layer main component.
- Keep adapters humble. Controllers, endpoints, presenters, gateway adapters, service listeners, and hardware adapters translate external formats to use-case calls and back; they do not own business decisions.
- Organize by use case, feature, or business capability before generic technical buckets. The structure should reveal domain intent and application actions.
- Choose boundaries by volatility, policy importance, substitution value, testability, and cost. Use the lightest enforceable boundary, including partial boundaries, when full deployment or runtime separation is too expensive.
- Do not merge unrelated use cases or eliminate duplication when sharing would couple actors, change reasons, team ownership, deployment needs, or release pressure.
- Use structured code, dependency inversion, role-sized interfaces, substitutable implementations, controlled mutation, acyclic components, and stability-directed dependencies to protect policy from volatile details.
- Enforce boundaries with package structure, dependency rules, build constraints, tests, visibility, or narrow APIs. A diagram, service split, package name, or shared `common` folder is not enough.
- Test entities, use cases, and boundary contracts first, without the real framework, database, network, external service, or target hardware. Test adapters separately at the seams.
- Preserve behavior while improving dependency direction. Prefer incremental boundary extraction over rewrites, and call out architectural debt when it cannot be fixed safely now.

## Trigger rules

- When urgent delivery would skip architecture, state the future change, test, replacement, or operational cost before accepting the shortcut.
- When framework annotations, request/response objects, serializers, ORM rows, schemas, vendor SDKs, config, environment reads, device registers, or transport formats enter core policy, move translation outward.
- When controllers, jobs, handlers, views, presenters, gateways, repositories, SQL, service listeners, scripts, or hardware adapters contain business branching or validation, move the rule inward.
- When a use case instantiates infrastructure, calls a volatile dependency directly, or depends on a concrete implementation, introduce a policy-owned port and wire the concrete detail at the edge.
- When a `*Service`, utility folder, shared module, base package, or generic `core` package becomes an escape hatch, split by use case, role, or ownership and restore dependency direction.
- When an adapter bypasses a use case, a presenter reads persistence directly, or infrastructure is both imported by and importing inward code, restore the intended boundary.
- When service boundaries, process boundaries, remote calls, deployment boundaries, or embedded hardware appear, still verify source dependencies, data ownership, I/O cost, and policy independence.
- When tests need the framework, database, network, service, or hardware to verify business rules, move tests to use cases/entities with fakes or add a stable boundary contract.
- When a compromise is unavoidable, keep it at the outermost layer possible, document the violation, avoid normalizing it, and preserve a path to separation.

## Final checklist

- Business rules independent from frameworks, databases, UI, services, devices, and vendors?
- Dependencies point inward, with ports owned by inner policy and concrete details outside?
- Entities guard invariants and focused use cases orchestrate one application action?
- Boundaries explicit and enforced in code, tests, packages, or build rules?
- Controllers, presenters, gateways, service listeners, and adapters humble?
- Structure reveals use cases and business capabilities instead of generic technical buckets?
- Core tests run fast without real delivery, persistence, network, external service, or hardware?
- Details remain replaceable without rewriting business rules?

---

# Supplément — Working Effectively with Legacy Code (source : ciembor/agent-rules-books)

# OBEY Working Effectively with Legacy Code by Michael Feathers

## When to use

Use when changing code that is expensive to change safely because behavior is unclear, tests are weak or missing, dependencies are hidden, or runtime/framework setup blocks local feedback.

## Primary bias to correct

Gain control before improving design. Understand current behavior, protect what must stay, create the smallest useful seam, break the dependency that blocks feedback, make the requested change, then leave the area more testable.

## Decision rules

- Treat any area without trustworthy tests as legacy code; do not start with rewrite or module-wide cleanup unless that is explicitly required or clearly safer.
- Before editing, state the requested behavior change and the current behavior that must remain; characterize uncertain or suspicious behavior instead of silently fixing it.
- Follow the legacy loop: identify the change point, check existing protection, add characterization where possible, find or create a seam, break the blocking dependency, change behavior, then refactor locally.
- Prefer fast, focused tests around the slice being changed; use broader interception or integration tests only when they are the safest first observation point.
- Choose test points by tracing effects outward from the change point through values, calls, fields, outputs, collaborators, interception points, and pinch points.
- Use the smallest seam that allows substitution, observation, or interception; make clear whether the seam is for sensing, separation, or both.
- Break dependencies deliberately: expose hidden inputs, hard outputs, hard construction, globals, statics, ambient context, and framework callbacks only where they block testing or safe change.
- Keep behavior changes, structural refactorings, and cleanup separate; verify small steps and avoid checking in exploratory restructuring used only for understanding.
- When direct edits are risky, add behavior with sprout method, sprout class, wrap method, wrap class, or extract-and-override style moves, then fold the temporary structure into better design when tests support it.
- For hard-to-test methods, split construction from use, extract side effects behind collaborators, carve pure computation first, and isolate policy from runtime, persistence, UI, or framework mechanisms.
- Use dependency-breaking techniques according to the actual barrier: adapt narrow parameters, extract interfaces or implementers, parameterize constructors or methods, encapsulate globals, introduce instance delegators, override factories/calls, or use link/preprocessing seams only when ordinary object seams are impractical.
- In large code, sketch effects and group responsibilities before moving behavior; let excessive setup, impossible observation, and repeated changes point to smaller extracted responsibilities.
- During review, treat no tests around modified logic, mixed structural and behavioral edits, broad edits in poorly understood modules, hard-coded collaborators, global/static reach-through, constructor side effects, and business logic trapped in framework entry points as legacy-change risks.
- Reject changes that expand hidden dependencies, mock around untestable structure without improving it, rename or format while leaving the real dependency knots intact, or introduce large architecture before basic seams exist.
- Leave the touched area easier to understand, test, or change; do not mistake test-only seams, wrappers, subclass tricks, or build tricks for design improvement by themselves.

## Trigger rules

- When behavior is uncertain, consumers may rely on ugly behavior, or a branch/path is hard to prove, add characterization or another explicit observation path before changing semantics.
- When tests require too much setup or a class cannot be instantiated cheaply, break the first real barrier: constructor work, hidden allocation, factory call, global state, static construction, framework object, or hard parameter.
- When time, randomness, environment, thread-local state, current user/request, files, network, process exits, database writes, messages, or control-flow logging block repeatable tests, wrap or inject that boundary.
- When a large method or class defeats local reasoning, sketch effects, find interception or pinch points, extract pure computation first, and avoid editing many branches at once.
- When changing database-heavy, UI, framework, or API-boundary code, separate policy from query/mapping/persistence, handlers/callbacks, adapters, and runtime setup; keep real-boundary integration tests where they matter.
- When a seam is magical, temporary, public-for-test, subclass-only, link/preprocessor-based, or probe/sensing-variable-based, add a cleanup obligation and remove it once safer structure exists.
- When repeated edits cluster across several places, remove duplication incrementally under tests instead of launching a broad redesign.
- When rewrite or heroic cleanup feels tempting, choose the smallest sprout, wrap, seam, characterization, or refactoring step that makes today's requested change safer.

## Final checklist

- Untested or weakly tested area treated as legacy risk?
- Behavior delta and behavior-to-preserve stated?
- Uncertain current behavior characterized or explicitly observed?
- Tests close enough and fast enough to diagnose the change?
- Smallest useful seam chosen, with sensing vs separation clear?
- Blocking dependency reduced without expanding hidden dependencies?
- Behavior change, refactoring, and cleanup kept separate?
- Temporary seam or dependency-breaking trick has a cleanup path?
- Touched area is more understandable, testable, or changeable?

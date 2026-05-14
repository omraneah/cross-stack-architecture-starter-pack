# Naming Conventions Boundaries

**Each layer has one naming convention. The convention is enforced automatically. No transformation layer sits between layers as long-term state.**

**Applies when:** always — any codebase with a persistence layer and an API surface.

## Why it matters

- Mixed naming conventions force every developer to translate mentally; the inconsistency creates serialization bugs at boundaries.
- Transformation interceptors that "convert" between conventions accumulate exceptions; the exception list becomes the actual contract.
- New naming patterns introduced by every new developer fork the codebase quietly.

## The judgment

**Application code:**

- Pick one: `camelCase` or `snake_case`. Both are defensible. Whichever you pick, apply it to variables, properties, parameters, and method names everywhere — backend, frontend, mobile.
- Types and classes follow language convention (typically `PascalCase`).

**API boundary:**

- One naming convention on request and response payloads, same as application code. No server-side transformation.
- Versioned APIs lock the convention; old unversioned routes may transform during migration, then disappear.

**Persistence:**

- Database columns and stored structures use a single convention. Common choice: `snake_case` for SQL persistence.
- Mapping between application convention and persistence convention happens in one explicit layer (entity or repository), not scattered.

**File naming:**

- Follow the language / framework convention for the project. Don't fight the ecosystem.

## Signals of violation in an audited codebase

- A new API endpoint returning a different naming convention than the rest of the API.
- A transformation interceptor with a growing exception list of routes it doesn't transform.
- Application-level constants or types using the persistence convention (or vice versa).
- File names within one project that follow two different conventions side by side.
- A linter that warns on naming violations but doesn't fail CI.

## Minimum viable shape

Five independent rules, one per layer:

- **Application layer:** one convention (`camelCase` or `snake_case`) used consistently.
- **API boundary:** same convention as the application; no transformation interceptors.
- **Persistence layer:** one convention; mapping to the application convention lives in the entity or repository, not scattered.
- **File naming:** follow the language / framework convention; don't fight the ecosystem.
- **Enforcement:** linter or formatter fails CI on violation; humans don't enforce naming.

**Severity floor if violated:** P2 — discipline drift, mostly cosmetic at small scale. Steps up to P1 in dense codebases where inconsistency costs mental translation cycles on every change, and to P0 if a transformation interceptor at the API boundary is masking a breaking-change to mobile clients.

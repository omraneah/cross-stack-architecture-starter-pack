# Naming Conventions Boundaries

**Each layer has one naming convention. The convention is enforced automatically. No transformation layer sits between layers as long-term state.**

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

```
Application layer → one convention consistently
API boundary → same as application; no transformation
Persistence layer → one convention; mapping in entity / repository only
File naming → language / framework convention
Enforcement → linter or formatter fails CI on violation
```

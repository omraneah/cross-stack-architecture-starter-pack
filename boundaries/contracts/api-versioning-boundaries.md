# API Versioning Boundaries

**Every public API endpoint has a stable, versioned contract. Breaking changes require a new version.**

## Why it matters

- Without versioning, every contract change forces simultaneous deploy of every client. App store delays make this fatal for mobile.
- Silent breaking changes (renamed fields, changed enums) surface as runtime errors days later, far from the change.
- Mixed unversioned and versioned routes make deprecation guesswork.

## The judgment

**Where the version lives:**

- URI path (`/api/v1/...`). Visible in logs, monitoring, and client code. Easiest to deprecate. Default for public-facing APIs and APIs consumed by independent clients.
- Header (`Accept-Version` or media type). Wins when the API is consumed by a tightly-coupled client SDK that handles versioning transparently, when content-negotiation is central to the design (HATEOAS, hypermedia-driven), or when the surface must keep clean URIs for SEO or aesthetics. Costlier to monitor and grep, so the trade-off has to justify itself.
- Query parameter. Discouraged — easy to forget, easy to cache wrong.

**Breaking-change protocol:**

- Always create a new version; keep the old version alive until traffic confirms zero usage.
- In-place edits to an existing version are forbidden, even for "obviously safe" renames.

**Naming convention at the boundary:**

- One convention (`camelCase` or `snake_case`) used consistently. No transformation layer at the API boundary.
- Transformation interceptors are technical debt — they fork the API surface and accumulate exceptions.

## Signals of violation in an audited codebase

- A controller without an explicit version declaration.
- A breaking change applied in place to an existing version's DTO.
- A test or client SDK base URL that omits the version segment.
- An unversioned route in production with no documented deprecation date or zero-traffic confirmation.
- Mixed naming conventions on the same response payload.

## Minimum viable shape

```
Controller declares version v1 explicitly
  → New endpoint or additive field? → stays in v1
  → Removal, rename, type change? → new v2 controller; v1 untouched
  → Tests, clients, docs all reference the versioned base URL
  → Deprecation = announce → monitor traffic → remove only at zero usage
```
